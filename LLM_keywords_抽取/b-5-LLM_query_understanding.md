可以。下面我直接给你一版**可上线取向的 v1**，默认前提是：

* 底层已经有 `keyword search` 和 `semantic search` 接口
* 已经有一个 **最小 alias 表**
* 数据规范化还不成熟，所以 **LLM 必须保守**
* LLM 模块只负责 **query understanding + retrieval planning 的中间表示**
* **不让 LLM 直接拼最终 ES DSL / 检索语句**

---

# 1. v1 设计原则

这版 v1 的核心目标不是“理解得多聪明”，而是：

1. **输出稳定**
2. **便于后处理**
3. **便于评测**
4. **便于回放日志**
5. **在 alias / normalization 不完善时尽量少犯大错**

所以这版设计里，LLM 要遵守一个很重要的原则：

> **宁可少做 normalization，也不要乱做 normalization。**

也就是：

* 命中 alias 表：可以规范化
* 没命中 alias 表，但用户 query 本身很明确：保留原词
* 不要擅自脑补一个 canonical ingredient / property

---

# 2. 建议的服务输入

先定义 **LLM query understanding service** 的输入，不然 prompt 不好落地。

```json
{
  "query": "我想找清爽、不泛白的物理防晒乳配方，最好是 ZnO / TiO2 体系",
  "language_hint": "zh",
  "domain": "materials_formulation",
  "alias_hints": [
    {
      "alias": "ZnO",
      "canonical": "zinc oxide",
      "type": "ingredient"
    },
    {
      "alias": "TiO2",
      "canonical": "titanium dioxide",
      "type": "ingredient"
    },
    {
      "alias": "防晒乳",
      "canonical": "sunscreen lotion",
      "type": "dosage_form"
    },
    {
      "alias": "清爽",
      "canonical": "non-greasy",
      "type": "property"
    },
    {
      "alias": "不泛白",
      "canonical": "low whitening",
      "type": "property"
    }
  ]
}
```

这里的 `alias_hints` 不是让 LLM 去“自由联想”，而是给它一个**可控的 normalization 提示集**。

---

# 3. v1 输出 schema

我建议 v1 先用这个结构。它不算大，但已经够支撑：

* 结构化理解
* normalization
* retrieval planner
* 离线 eval
* 线上回放

```json
{
  "schema_version": "qu_v1",
  "query_text": "string",
  "query_language": "zh|en|mixed",
  "intent": {
    "primary": "formulation_lookup|ingredient_lookup|property_lookup|process_lookup|comparison|troubleshooting|exploratory|other",
    "confidence": 0.0
  },
  "entities": [
    {
      "mention": "string",
      "normalized": "string",
      "type": "ingredient|function|property|dosage_form|application|process|material_class|other",
      "role": "core|constraint|expansion|exclude",
      "normalization_status": "exact_alias|self|likely_alias|unmapped",
      "confidence": 0.0
    }
  ],
  "constraints": {
    "applications": ["string"],
    "dosage_forms": ["string"],
    "include_terms": ["string"],
    "exclude_terms": ["string"],
    "property_targets": ["string"],
    "process_terms": ["string"],
    "numeric_constraints": [
      {
        "attribute": "string",
        "operator": ">|>=|<|<=|=|range",
        "value": "string",
        "unit": "string"
      }
    ]
  },
  "retrieval_plan": {
    "mode": "keyword_only|semantic_only|hybrid",
    "must_terms": ["string"],
    "should_terms": ["string"],
    "exclude_terms": ["string"],
    "filters": {
      "application": ["string"],
      "dosage_form": ["string"],
      "material_class": ["string"]
    },
    "semantic_query": "string"
  },
  "clarification": {
    "needed": false,
    "reason": "string",
    "question": "string"
  }
}
```

---

## 3.1 字段解释

### `intent.primary`

表示主意图。v1 够用的几类：

* `formulation_lookup`：找配方/体系
* `ingredient_lookup`：找成分
* `property_lookup`：找性能/属性相关结果
* `process_lookup`：找工艺/步骤
* `comparison`：A vs B
* `troubleshooting`：为什么会这样、如何改进
* `exploratory`：模糊探索
* `other`

### `entities`

这是 LLM 的主要中间表示。

例如：

```json
{
  "mention": "ZnO",
  "normalized": "zinc oxide",
  "type": "ingredient",
  "role": "core",
  "normalization_status": "exact_alias",
  "confidence": 0.99
}
```

这里的关键是 `normalization_status`：

* `exact_alias`：来自 alias 表，最可信
* `self`：没有做映射，保留原词
* `likely_alias`：模型推测可能是别名，但不够硬
* `unmapped`：识别了 mention，但没有可用规范化

对于 v1，我建议程序侧可以这样用：

* `exact_alias`：可放心进 keyword must/should
* `self`：可以保留
* `likely_alias`：尽量只进 semantic query，不进 must
* `unmapped`：保守处理

### `constraints`

把结构化约束单独放出来，方便 planner。

### `retrieval_plan`

这是给后面的 search 层吃的，不是最终 ES DSL，但已经足够接近“可执行计划”。

### `clarification`

v1 不建议真的卡住用户。
即使 `needed = true`，也仍然要输出一个 best-effort plan。

---

# 4. v1 的 system prompt

下面这版是可以直接拿去落地的。

---

```text
You are a query-understanding engine for a materials / formulation search system.

Your job is NOT to answer the user.
Your job is to convert the user's natural language query into a conservative, structured retrieval plan.

You must output JSON only.
Do not output markdown.
Do not output explanations.
Do not output any text outside the JSON object.

You will receive:
1) query: the raw user query
2) language_hint: one of zh, en, mixed
3) domain: usually materials_formulation
4) alias_hints: a small list of trusted alias mappings detected by upstream rules

Your goals:
- identify the main search intent
- extract entities and constraints from the query
- normalize terms conservatively
- produce a retrieval_plan for keyword search + semantic search
- indicate whether clarification is needed

Important normalization rules:
- Be conservative.
- Only normalize confidently.
- Prefer alias_hints whenever available.
- If a query term matches an alias_hints entry, use its canonical value and set normalization_status = "exact_alias".
- If there is no trusted alias match, do NOT invent a canonical chemical or property term.
- If a term is clear in the query but not mapped, keep normalized equal to the original mention and set normalization_status = "self" or "unmapped".
- Use "likely_alias" only when the mapping is strongly suggested by the query wording, but not directly supported by alias_hints.
- Never hallucinate ingredients, formulations, dosage forms, applications, or process names that are not present in the query or alias_hints.

Entity typing rules:
- ingredient: chemical, material, additive, polymer, active, pigment, solvent, surfactant, etc.
- function: functional role such as humectant, dispersant, film-forming, emulsifier
- property: target property such as non-greasy, low whitening, adhesion, water resistance, stability
- dosage_form: lotion, cream, gel, spray, emulsion, coating, etc.
- application: sunscreen, hair care, skin care, coating, adhesive, oral care, etc.
- process: mixing, dispersion, polymerization, curing, drying, encapsulation, etc.
- material_class: silicone-free, acrylic, polyurethane, inorganic UV filter, etc.
- other: anything important but not covered above

Role rules:
- core: central concept of the query
- constraint: required condition
- expansion: useful related concept that may help retrieval
- exclude: explicitly unwanted concept

Intent rules:
- formulation_lookup: user wants formulas, systems, compositions, recipes, embodiments
- ingredient_lookup: user wants ingredient-focused information
- property_lookup: user wants property/performance-focused results
- process_lookup: user wants process/manufacturing/preparation method
- comparison: user compares two or more alternatives
- troubleshooting: user describes a problem and seeks causes or fixes
- exploratory: user asks vaguely or incompletely
- other: none of the above

Retrieval planning rules:
- retrieval_plan.mode should usually be "hybrid" unless the query is obviously only keyword-oriented or only semantic-oriented
- must_terms: indispensable terms, usually <= 6
- should_terms: useful expansions, usually <= 8
- exclude_terms: unwanted terms, usually <= 6
- semantic_query: a short natural-language retrieval query, not Boolean syntax
- filters.application and filters.dosage_form should only contain stable, high-confidence constraints

Clarification rules:
- If the query is underspecified or ambiguous, set clarification.needed = true
- Still produce the best possible retrieval_plan
- clarification.question should be short and practical

Confidence rules:
- confidence is a float between 0 and 1
- use higher confidence only when strongly supported by the query or alias_hints

Return JSON with exactly this structure:

{
  "schema_version": "qu_v1",
  "query_text": "...",
  "query_language": "zh|en|mixed",
  "intent": {
    "primary": "formulation_lookup|ingredient_lookup|property_lookup|process_lookup|comparison|troubleshooting|exploratory|other",
    "confidence": 0.0
  },
  "entities": [
    {
      "mention": "...",
      "normalized": "...",
      "type": "ingredient|function|property|dosage_form|application|process|material_class|other",
      "role": "core|constraint|expansion|exclude",
      "normalization_status": "exact_alias|self|likely_alias|unmapped",
      "confidence": 0.0
    }
  ],
  "constraints": {
    "applications": [],
    "dosage_forms": [],
    "include_terms": [],
    "exclude_terms": [],
    "property_targets": [],
    "process_terms": [],
    "numeric_constraints": []
  },
  "retrieval_plan": {
    "mode": "keyword_only|semantic_only|hybrid",
    "must_terms": [],
    "should_terms": [],
    "exclude_terms": [],
    "filters": {
      "application": [],
      "dosage_form": [],
      "material_class": []
    },
    "semantic_query": "..."
  },
  "clarification": {
    "needed": false,
    "reason": "",
    "question": ""
  }
}
```

---

# 5. few-shot 示例

下面给你 5 条。我建议 v1 先用 4~5 条，不要太多。

---

## Few-shot 1：中文，配方检索，带 ingredient + property + dosage form

### Input

```json
{
  "query": "我想找清爽、不泛白的物理防晒乳配方，最好是 ZnO / TiO2 体系",
  "language_hint": "zh",
  "domain": "materials_formulation",
  "alias_hints": [
    {
      "alias": "ZnO",
      "canonical": "zinc oxide",
      "type": "ingredient"
    },
    {
      "alias": "TiO2",
      "canonical": "titanium dioxide",
      "type": "ingredient"
    },
    {
      "alias": "防晒乳",
      "canonical": "sunscreen lotion",
      "type": "dosage_form"
    },
    {
      "alias": "清爽",
      "canonical": "non-greasy",
      "type": "property"
    },
    {
      "alias": "不泛白",
      "canonical": "low whitening",
      "type": "property"
    }
  ]
}
```

### Output

```json
{
  "schema_version": "qu_v1",
  "query_text": "我想找清爽、不泛白的物理防晒乳配方，最好是 ZnO / TiO2 体系",
  "query_language": "zh",
  "intent": {
    "primary": "formulation_lookup",
    "confidence": 0.96
  },
  "entities": [
    {
      "mention": "防晒乳",
      "normalized": "sunscreen lotion",
      "type": "dosage_form",
      "role": "core",
      "normalization_status": "exact_alias",
      "confidence": 0.98
    },
    {
      "mention": "清爽",
      "normalized": "non-greasy",
      "type": "property",
      "role": "constraint",
      "normalization_status": "exact_alias",
      "confidence": 0.95
    },
    {
      "mention": "不泛白",
      "normalized": "low whitening",
      "type": "property",
      "role": "constraint",
      "normalization_status": "exact_alias",
      "confidence": 0.95
    },
    {
      "mention": "ZnO",
      "normalized": "zinc oxide",
      "type": "ingredient",
      "role": "core",
      "normalization_status": "exact_alias",
      "confidence": 0.99
    },
    {
      "mention": "TiO2",
      "normalized": "titanium dioxide",
      "type": "ingredient",
      "role": "core",
      "normalization_status": "exact_alias",
      "confidence": 0.99
    },
    {
      "mention": "物理防晒",
      "normalized": "物理防晒",
      "type": "application",
      "role": "core",
      "normalization_status": "self",
      "confidence": 0.74
    }
  ],
  "constraints": {
    "applications": ["物理防晒"],
    "dosage_forms": ["sunscreen lotion"],
    "include_terms": ["zinc oxide", "titanium dioxide"],
    "exclude_terms": [],
    "property_targets": ["non-greasy", "low whitening"],
    "process_terms": [],
    "numeric_constraints": []
  },
  "retrieval_plan": {
    "mode": "hybrid",
    "must_terms": ["sunscreen lotion", "zinc oxide", "titanium dioxide"],
    "should_terms": ["non-greasy", "low whitening", "物理防晒"],
    "exclude_terms": [],
    "filters": {
      "application": [],
      "dosage_form": ["sunscreen lotion"],
      "material_class": []
    },
    "semantic_query": "physical sunscreen lotion formulation with zinc oxide and titanium dioxide, non-greasy feel, low whitening"
  },
  "clarification": {
    "needed": false,
    "reason": "",
    "question": ""
  }
}
```

---

## Few-shot 2：英文，coating 场景，带 exclusion

### Input

```json
{
  "query": "Find a water-based acrylic coating with good adhesion and fast drying, avoid solvent-borne systems",
  "language_hint": "en",
  "domain": "materials_formulation",
  "alias_hints": [
    {
      "alias": "water-based",
      "canonical": "waterborne",
      "type": "material_class"
    },
    {
      "alias": "fast drying",
      "canonical": "fast drying",
      "type": "property"
    },
    {
      "alias": "good adhesion",
      "canonical": "adhesion",
      "type": "property"
    },
    {
      "alias": "solvent-borne",
      "canonical": "solventborne",
      "type": "material_class"
    }
  ]
}
```

### Output

```json
{
  "schema_version": "qu_v1",
  "query_text": "Find a water-based acrylic coating with good adhesion and fast drying, avoid solvent-borne systems",
  "query_language": "en",
  "intent": {
    "primary": "formulation_lookup",
    "confidence": 0.95
  },
  "entities": [
    {
      "mention": "water-based",
      "normalized": "waterborne",
      "type": "material_class",
      "role": "constraint",
      "normalization_status": "exact_alias",
      "confidence": 0.97
    },
    {
      "mention": "acrylic",
      "normalized": "acrylic",
      "type": "material_class",
      "role": "core",
      "normalization_status": "self",
      "confidence": 0.93
    },
    {
      "mention": "coating",
      "normalized": "coating",
      "type": "application",
      "role": "core",
      "normalization_status": "self",
      "confidence": 0.95
    },
    {
      "mention": "good adhesion",
      "normalized": "adhesion",
      "type": "property",
      "role": "constraint",
      "normalization_status": "exact_alias",
      "confidence": 0.96
    },
    {
      "mention": "fast drying",
      "normalized": "fast drying",
      "type": "property",
      "role": "constraint",
      "normalization_status": "exact_alias",
      "confidence": 0.96
    },
    {
      "mention": "solvent-borne",
      "normalized": "solventborne",
      "type": "material_class",
      "role": "exclude",
      "normalization_status": "exact_alias",
      "confidence": 0.98
    }
  ],
  "constraints": {
    "applications": ["coating"],
    "dosage_forms": [],
    "include_terms": ["waterborne", "acrylic"],
    "exclude_terms": ["solventborne"],
    "property_targets": ["adhesion", "fast drying"],
    "process_terms": [],
    "numeric_constraints": []
  },
  "retrieval_plan": {
    "mode": "hybrid",
    "must_terms": ["coating", "waterborne", "acrylic"],
    "should_terms": ["adhesion", "fast drying"],
    "exclude_terms": ["solventborne"],
    "filters": {
      "application": ["coating"],
      "dosage_form": [],
      "material_class": ["waterborne", "acrylic"]
    },
    "semantic_query": "waterborne acrylic coating formulation with good adhesion and fast drying, excluding solventborne systems"
  },
  "clarification": {
    "needed": false,
    "reason": "",
    "question": ""
  }
}
```

---

## Few-shot 3：中英混合，troubleshooting

### Input

```json
{
  "query": "为什么含 PVP 的 hair spray 容易发黏？想找 anti-tack / low tack 的成膜体系",
  "language_hint": "mixed",
  "domain": "materials_formulation",
  "alias_hints": [
    {
      "alias": "PVP",
      "canonical": "polyvinylpyrrolidone",
      "type": "ingredient"
    },
    {
      "alias": "hair spray",
      "canonical": "hair spray",
      "type": "dosage_form"
    },
    {
      "alias": "anti-tack",
      "canonical": "low tack",
      "type": "property"
    },
    {
      "alias": "low tack",
      "canonical": "low tack",
      "type": "property"
    },
    {
      "alias": "成膜",
      "canonical": "film-forming",
      "type": "function"
    }
  ]
}
```

### Output

```json
{
  "schema_version": "qu_v1",
  "query_text": "为什么含 PVP 的 hair spray 容易发黏？想找 anti-tack / low tack 的成膜体系",
  "query_language": "mixed",
  "intent": {
    "primary": "troubleshooting",
    "confidence": 0.94
  },
  "entities": [
    {
      "mention": "PVP",
      "normalized": "polyvinylpyrrolidone",
      "type": "ingredient",
      "role": "core",
      "normalization_status": "exact_alias",
      "confidence": 0.99
    },
    {
      "mention": "hair spray",
      "normalized": "hair spray",
      "type": "dosage_form",
      "role": "core",
      "normalization_status": "exact_alias",
      "confidence": 0.98
    },
    {
      "mention": "发黏",
      "normalized": "发黏",
      "type": "property",
      "role": "constraint",
      "normalization_status": "self",
      "confidence": 0.81
    },
    {
      "mention": "anti-tack",
      "normalized": "low tack",
      "type": "property",
      "role": "constraint",
      "normalization_status": "exact_alias",
      "confidence": 0.96
    },
    {
      "mention": "low tack",
      "normalized": "low tack",
      "type": "property",
      "role": "constraint",
      "normalization_status": "exact_alias",
      "confidence": 0.97
    },
    {
      "mention": "成膜",
      "normalized": "film-forming",
      "type": "function",
      "role": "core",
      "normalization_status": "exact_alias",
      "confidence": 0.95
    }
  ],
  "constraints": {
    "applications": [],
    "dosage_forms": ["hair spray"],
    "include_terms": ["polyvinylpyrrolidone", "film-forming"],
    "exclude_terms": [],
    "property_targets": ["low tack"],
    "process_terms": [],
    "numeric_constraints": []
  },
  "retrieval_plan": {
    "mode": "hybrid",
    "must_terms": ["hair spray", "polyvinylpyrrolidone", "film-forming"],
    "should_terms": ["low tack", "anti-tack", "发黏"],
    "exclude_terms": [],
    "filters": {
      "application": [],
      "dosage_form": ["hair spray"],
      "material_class": []
    },
    "semantic_query": "low tack film-forming system for hair spray containing polyvinylpyrrolidone, addressing tackiness"
  },
  "clarification": {
    "needed": false,
    "reason": "",
    "question": ""
  }
}
```

---

## Few-shot 4：comparison query

### Input

```json
{
  "query": "compare silicone-free conditioner formulations using cationic guar vs polyquaternium",
  "language_hint": "en",
  "domain": "materials_formulation",
  "alias_hints": [
    {
      "alias": "silicone-free",
      "canonical": "silicone-free",
      "type": "material_class"
    },
    {
      "alias": "conditioner",
      "canonical": "conditioner",
      "type": "dosage_form"
    },
    {
      "alias": "cationic guar",
      "canonical": "cationic guar",
      "type": "ingredient"
    },
    {
      "alias": "polyquaternium",
      "canonical": "polyquaternium",
      "type": "ingredient"
    }
  ]
}
```

### Output

```json
{
  "schema_version": "qu_v1",
  "query_text": "compare silicone-free conditioner formulations using cationic guar vs polyquaternium",
  "query_language": "en",
  "intent": {
    "primary": "comparison",
    "confidence": 0.98
  },
  "entities": [
    {
      "mention": "silicone-free",
      "normalized": "silicone-free",
      "type": "material_class",
      "role": "constraint",
      "normalization_status": "exact_alias",
      "confidence": 0.98
    },
    {
      "mention": "conditioner",
      "normalized": "conditioner",
      "type": "dosage_form",
      "role": "core",
      "normalization_status": "exact_alias",
      "confidence": 0.99
    },
    {
      "mention": "cationic guar",
      "normalized": "cationic guar",
      "type": "ingredient",
      "role": "core",
      "normalization_status": "exact_alias",
      "confidence": 0.98
    },
    {
      "mention": "polyquaternium",
      "normalized": "polyquaternium",
      "type": "ingredient",
      "role": "core",
      "normalization_status": "exact_alias",
      "confidence": 0.98
    }
  ],
  "constraints": {
    "applications": [],
    "dosage_forms": ["conditioner"],
    "include_terms": ["silicone-free", "cationic guar", "polyquaternium"],
    "exclude_terms": [],
    "property_targets": [],
    "process_terms": [],
    "numeric_constraints": []
  },
  "retrieval_plan": {
    "mode": "hybrid",
    "must_terms": ["conditioner", "silicone-free"],
    "should_terms": ["cationic guar", "polyquaternium", "comparison"],
    "exclude_terms": [],
    "filters": {
      "application": [],
      "dosage_form": ["conditioner"],
      "material_class": ["silicone-free"]
    },
    "semantic_query": "comparison of silicone-free conditioner formulations using cationic guar versus polyquaternium"
  },
  "clarification": {
    "needed": false,
    "reason": "",
    "question": ""
  }
}
```

---

## Few-shot 5：模糊探索，需要 clarification，但仍给 plan

### Input

```json
{
  "query": "想找更稳定的乳液体系",
  "language_hint": "zh",
  "domain": "materials_formulation",
  "alias_hints": [
    {
      "alias": "乳液",
      "canonical": "emulsion",
      "type": "dosage_form"
    },
    {
      "alias": "稳定",
      "canonical": "stability",
      "type": "property"
    }
  ]
}
```

### Output

```json
{
  "schema_version": "qu_v1",
  "query_text": "想找更稳定的乳液体系",
  "query_language": "zh",
  "intent": {
    "primary": "exploratory",
    "confidence": 0.89
  },
  "entities": [
    {
      "mention": "乳液",
      "normalized": "emulsion",
      "type": "dosage_form",
      "role": "core",
      "normalization_status": "exact_alias",
      "confidence": 0.98
    },
    {
      "mention": "稳定",
      "normalized": "stability",
      "type": "property",
      "role": "constraint",
      "normalization_status": "exact_alias",
      "confidence": 0.93
    },
    {
      "mention": "体系",
      "normalized": "体系",
      "type": "other",
      "role": "core",
      "normalization_status": "self",
      "confidence": 0.62
    }
  ],
  "constraints": {
    "applications": [],
    "dosage_forms": ["emulsion"],
    "include_terms": [],
    "exclude_terms": [],
    "property_targets": ["stability"],
    "process_terms": [],
    "numeric_constraints": []
  },
  "retrieval_plan": {
    "mode": "hybrid",
    "must_terms": ["emulsion"],
    "should_terms": ["stability", "stable emulsion"],
    "exclude_terms": [],
    "filters": {
      "application": [],
      "dosage_form": ["emulsion"],
      "material_class": []
    },
    "semantic_query": "stable emulsion formulation system"
  },
  "clarification": {
    "needed": true,
    "reason": "application, ingredient family, and target scenario are not specified",
    "question": "是想找哪类应用的乳液体系，比如护肤、涂料还是其他场景？"
  }
}
```

---

# 6. 这版 v1 为什么适合你当前阶段

你现在的前提是：

* 数据规范化不明
* alias 表还很小
* search 接口已经有
* 现在最需要的是一个**可验证、可迭代**的 query understanding 模块

这版设计的优点是：

### 1）保守

不会因为 LLM“太聪明”而乱扩词。

### 2）可接后处理

后面程序可以非常明确地做：

* schema validate
* dedupe
* confidence trim
* alias overwrite
* retrieval plan builder

### 3）可评测

你后面完全可以针对下面这些字段做 gold 标注：

* `intent.primary`
* `entities`
* `normalization_status`
* `must_terms / should_terms`
* `clarification.needed`

### 4）适合 data normalization 还没成熟的阶段

因为它允许：

* 命中 alias 的强规范化
* 未命中的保守保留原词

这个比“强行让 LLM 全部规范化”为好。

---

# 7. v1 上线前，程序侧至少再补这 4 个后处理规则

虽然你这次主要要的是 schema + prompt + few-shot，但这 4 个我建议同步做，不然第一版会有噪声。

### 规则 1：只信任 `exact_alias`

* `exact_alias` 可直接进 keyword 检索
* `likely_alias` 默认只进 semantic query
* `unmapped` 不做硬 filter

### 规则 2：裁剪 terms 数量

建议：

* `must_terms <= 6`
* `should_terms <= 8`
* `exclude_terms <= 6`

### 规则 3：去重

比如 `anti-tack` 和 `low tack` 都映射到 `low tack` 时，keyword 侧要去重。

### 规则 4：clarification 不阻塞

`clarification.needed = true` 时仍然执行检索，只是给前端一个可选追问。

---

# 8. 你下一步最合理的实现顺序

按工程顺序，我建议是：

先把这几个文件落下来：

```text
query_understanding/
├── schemas/
│   └── qu_v1_schema.json
├── prompts/
│   ├── system_prompt_v1.txt
│   └── fewshot_v1.json
├── service/
│   └── query_understanding_service.py
├── normalizers/
│   └── alias_table_v1.json
└── planner/
    └── retrieval_plan_postprocess.py
```

然后跑这个最小闭环：

1. 输入 query + alias_hints
2. 调 LLM
3. 校验 schema
4. 后处理
5. 生成 keyword_query / semantic_query
6. 并发调两个 search 接口
7. 打日志

---


