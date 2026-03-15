

可以。

你这里的 **5. query rewrite / retrieval planner**，本质上是把：

`用户自然语言`
先变成
`结构化检索意图`
再变成
`真正可执行的检索计划`

也就是两层：

1. **query rewrite**：把“人话”改写成适合检索系统的查询表达
2. **retrieval planner**：决定“该怎么搜、搜哪几路、每一路怎么配”

这两层，恰恰是把 **LLM 输出** 和 **现有 keyword / semantic search 接口** 接起来的关键中间层。

另外，你前面那种“先让 LLM 产结构化 JSON，再由程序消费”的思路是对的。OpenAI 官方文档里，Structured Outputs 的定位就是让模型按你给定的 JSON Schema 稳定输出；Prompt 指南也明确强调，复杂任务要靠清晰指令和清晰输出格式；而 eval 文档强调这种模块应该被当成可评测系统组件，而不是只凭感觉调 prompt。([developers.openai.com][1])

---

## 一、先用一句话说清楚两者区别

### 1）query rewrite 是什么

它解决的是：

> “用户说的话，不等于检索系统最该搜的话。”

比如用户说：

> 我想找清爽、不泛白的物理防晒乳配方

检索系统未必最擅长直接搜这句原话。
你往往要把它改写成几种更适合搜索的表达：

* 给 keyword search 的表达：

  * `physical sunscreen`
  * `lotion OR emulsion`
  * `zinc oxide OR titanium dioxide`
  * `non-whitening OR transparent`
  * `non-greasy`

* 给 semantic search 的表达：

  * `physical sunscreen lotion formulation with inorganic UV filters, low whitening, transparent appearance, non-greasy skin feel`

这就叫 **rewrite**。

它不是“随便改写一句更通顺的话”，而是：

> 为不同 retrieval backend 生成最合适的检索表达。

---

### 2）retrieval planner 是什么

它解决的是：

> “面对这个 query，这次检索到底该怎么打？”

比如同样是一个 query，planner 要决定：

* 要不要同时查 keyword + semantic
* 哪些词必须进 must
* 哪些词只能做 should expansion
* 要不要下 filters
* semantic query 用原句还是改写句
* 这次要不要做 query expansion
* 结果融合用什么策略
* 什么时候降级，什么时候跳过 LLM

所以 planner 更像一个 **检索策略控制器 / policy engine**。

---

## 二、一个更准确的理解

你可以这样理解整个前链路：

```text
User Query
  -> Query Understanding (LLM)
  -> Structured Intent
  -> Normalization
  -> Query Rewrite
  -> Retrieval Planner
  -> Execute keyword search + semantic search
  -> Fusion / rerank
```

其中：

### Query Understanding

负责“理解”

* 意图
* 实体
* 属性
* 约束
* 模糊词含义

### Query Rewrite

负责“表达转换”

* 原始 query -> keyword query
* 原始 query -> semantic query
* 原始 query -> filters / facets / boosts

### Retrieval Planner

负责“策略决策”

* 查哪些索引
* 每个索引怎么查
* 参数怎么配
* 哪些约束用于召回，哪些只用于 rerank
* 是否降级 / fallback

---

# 三、为什么不能只做 rewrite，不做 planner

因为 rewrite 只回答：

> “搜什么词？”

但 planner 还要回答：

> “怎么搜这些词？”

举个例子。

用户 query：

> 找用于透明喷雾型防晒的 ZnO 分散体系

即使 rewrite 已经做得很好，planner 仍然要决定：

* keyword 是否把 `transparent` 放 must
* `spray` 是 hard filter 还是 should
* `ZnO` 是否扩展 `zinc oxide`
* semantic 是否用长句描述
* process 词 `dispersion` 是否进 keyword
* 如果 keyword 太严格导致 0 召回，要不要自动放松

这些都不是 rewrite 本身能解决的。

所以：

* **rewrite = 生成候选检索表达**
* **planner = 选择执行策略**

---

# 四、在工程上，这两层各自应该怎么设计

---

## 1. Query Rewrite：工程职责

推荐把 rewrite 设计成一个**纯函数型模块**：

输入：

```json
{
  "raw_query": "我想找清爽不泛白的物理防晒乳配方",
  "understanding": {
    "intent": "lookup_formulation",
    "normalized_terms": [
      {"canonical": "physical sunscreen", "type": "function"},
      {"canonical": "lotion", "type": "dosage_form"},
      {"canonical": "non-greasy", "type": "property"},
      {"canonical": "low whitening", "type": "property"}
    ],
    "filters": {
      "application": ["sun care"],
      "dosage_form": ["lotion", "emulsion"]
    }
  }
}
```

输出：

```json
{
  "keyword_rewrites": [
    {
      "query": "\"physical sunscreen\" AND (lotion OR emulsion) AND (\"non-greasy\" OR \"low whitening\")",
      "weight": 1.0,
      "purpose": "balanced"
    },
    {
      "query": "(\"zinc oxide\" OR ZnO OR \"titanium dioxide\" OR TiO2) AND (lotion OR emulsion)",
      "weight": 0.8,
      "purpose": "ingredient_expansion"
    }
  ],
  "semantic_rewrites": [
    {
      "query": "physical sunscreen lotion formulation with inorganic UV filters, low whitening, transparent appearance, non-greasy skin feel",
      "weight": 1.0,
      "purpose": "dense_main"
    }
  ],
  "filters": {
    "application": ["sun care"],
    "dosage_form": ["lotion", "emulsion"]
  }
}
```

这里关键点是：

### rewrite 不要只产 1 条 query

建议产 **多条候选 rewrite**
因为不同 rewrite 适合不同召回路径：

* balanced rewrite
* precision rewrite
* expansion rewrite
* semantic summary rewrite

---

## 2. Retrieval Planner：工程职责

planner 则吃 rewrite 的结果，再生成真正的执行 plan。

输入：

```json
{
  "query_understanding": {...},
  "rewrites": {...},
  "runtime_context": {
    "llm_enabled": true,
    "latency_budget_ms": 1200,
    "search_backends": ["keyword", "semantic"],
    "query_length": 12
  }
}
```

输出：

```json
{
  "plan_type": "hybrid_parallel",
  "keyword_requests": [
    {
      "query": "\"physical sunscreen\" AND (lotion OR emulsion) AND (\"non-greasy\" OR \"low whitening\")",
      "top_k": 100,
      "filters": {
        "application": ["sun care"],
        "dosage_form": ["lotion", "emulsion"]
      },
      "must_mode": true
    }
  ],
  "semantic_requests": [
    {
      "query": "physical sunscreen lotion formulation with inorganic UV filters, low whitening, transparent appearance, non-greasy skin feel",
      "top_k": 100
    }
  ],
  "fusion": {
    "method": "rrf",
    "keyword_weight": 1.0,
    "semantic_weight": 1.0
  },
  "rerank": {
    "enabled": true,
    "top_n": 50
  },
  "fallback": {
    "on_zero_keyword_hit": "relax_keyword_constraints",
    "on_llm_failure": "use_raw_query"
  }
}
```

这才是“可执行计划”。

---

# 五、从系统设计上看，planner 其实是一个策略层

我建议你把 planner 理解成三部分：

---

## A. 路由决策（Routing）

决定这次请求走哪条路。

例如：

### 规则 1：极短 query，不走 LLM

* `ZnO`
* `TiO2 sunscreen`
* `US1234567`

这种直接：

* keyword 直查
* semantic 原 query 直查
* 不做复杂 rewrite

### 规则 2：自然语言复杂需求，走 LLM + planner

* 清爽不泛白的物理防晒乳
* 适合敏感肌的低刺激乳化体系
* 想找透明喷雾型防晒的无机分散方案

这种走完整链路。

### 规则 3：明确结构化约束，强 filter

* ingredient + dosage_form + property 三者都出现
* planner 下更强的 filters 和 must clauses

---

## B. 参数决策（Search Policy）

决定每一路搜索参数。

例如：

### keyword search

* top_k = 200 还是 50
* 用 strict must 还是 relaxed should
* 是否启用 synonym expansion
* 是否下推 filters
* 是否做 query relaxation

### semantic search

* 用原 query embedding 还是 rewrite query embedding
* top_k 取多少
* 是否多 query 并发召回
* 是否启用 expansion semantic rewrite

---

## C. 融合决策（Fusion Policy）

决定最终结果怎么合。

例如：

* keyword-only
* semantic-only
* hybrid parallel
* semantic first, keyword补召回
* keyword first, semantic补拓展

v1 通常最稳的是：

* **keyword + semantic 并发**
* 然后 **RRF 或简单加权融合**
* 再做 rerank

---

# 六、一个很重要的工程原则

## LLM 不要直接输出最终 DSL，最好输出“中间表示”

也就是：

### 不推荐 v1：

LLM 直接输出 ES DSL / Solr query / Vespa YQL

因为：

* 不稳定
* 难校验
* 难做版本演进
* 模型容易产非法结构

### 推荐 v1：

LLM 输出结构化中间表示
程序去拼 search request

例如让 LLM 输出：

```json
{
  "must_terms": ["physical sunscreen", "lotion"],
  "should_terms": ["zinc oxide", "titanium dioxide", "non-greasy", "low whitening"],
  "filters": {
    "application": ["sun care"],
    "dosage_form": ["lotion", "emulsion"]
  },
  "query_style": "balanced"
}
```

然后你程序里：

* `keyword_builder.py`
* `semantic_builder.py`
* `planner.py`

分别消费这份中间结果。

这个思路和 Structured Outputs 非常一致：先让模型稳定地产出符合 schema 的结构化结果，再由后续代码处理。([developers.openai.com][1])

---

# 七、retrieval planner 在你这个场景里的具体职责

你现在有：

* keyword search 接口
* semantic search 接口
* LLM API

那么 planner 最实际的职责就是这 6 个：

### 1. 选检索模式

* keyword-only
* semantic-only
* hybrid-parallel
* hybrid-with-relaxation

### 2. 分配 terms

哪些词进：

* keyword must
* keyword should
* semantic query
* rerank features

### 3. 下推 filters

哪些约束适合作为 hard filters：

* application
* dosage_form
* document_type
* year
* authority

### 4. 控制 query expansion

哪些词允许扩展：

* ZnO -> zinc oxide
* TiO2 -> titanium dioxide
* lotion -> emulsion

哪些不允许乱扩展：

* 具体 ingredient 名
* patent id
* 明确品牌词
* 用户给出的排除词

### 5. 控制召回宽严

* 先严格查
* 召回少时自动放松
* 放松顺序怎么定义

### 6. 定义 fallback

* LLM 超时怎么办
* JSON 无效怎么办
* rewrite 太激进怎么办
* 某一路 0 hit 怎么办

---

# 八、一个非常实用的 v1 planner 规则体系

你完全可以先不用 ML planner，直接做 **rule-based planner v1**。

---

## Case 1：精确实体查询

例子：

* `ZnO sunscreen`
* `polyvinylpyrrolidone film forming`
* `WO2020123456`

策略：

* keyword 权重大
* semantic 只作为补召回
* 少做 expansion
* 严格 must

```python
if is_precise_entity_query(q):
    mode = "hybrid_parallel"
    keyword.strict = True
    semantic.use_raw_query = True
    expansion.level = "low"
```

---

## Case 2：属性驱动型自然语言查询

例子：

* 清爽不泛白的物理防晒乳
* 透明、防水、低刺激的喷雾体系

策略：

* semantic 权重大
* keyword 保留核心短语
* 属性词不要全进 must
* 多数属性先进 should / rerank

```python
if is_property_driven_query(q):
    mode = "hybrid_parallel"
    keyword.must = core_entities_only
    keyword.should = properties
    semantic.use_rewrite = True
    rerank.use_constraints = True
```

---

## Case 3：模糊探索型查询

例子：

* 找一个更好的分散体系
* 有没有适合敏感肌的方案

策略：

* semantic 主召回
* keyword 只保留少量核心词
* planner 标注 `needs_clarification=true`
* 但仍给 best-effort plan

```python
if is_exploratory_query(q):
    mode = "semantic_heavy"
    keyword.query = minimal_core_terms
    semantic.query = abstract_need_rewrite
```

---

## Case 4：约束过多的查询

例子：

* 透明 + 防水 + 喷雾 + ZnO + 敏感肌 + 低刺激 + 不泛白

策略：

* 不能所有约束都进 keyword must
* planner 要决定“哪些用于召回，哪些用于 rerank”

这一步特别重要。

经验上：

### 进入召回层的：

* ingredient
* dosage_form
* application
* 大类功能词

### 留在 rerank 层的：

* 体验型属性
* 软性偏好
* 模糊性能词

否则很容易 0 recall。

---

# 九、工程上应该怎么落地

我建议你直接做 4 个模块。

---

## 模块 1：Query Understanding Service

职责：

* 调 LLM
* 输出结构化 JSON
* schema 校验
* normalization

接口：

```python
class QueryUnderstandingResult(BaseModel):
    intent: str
    core_terms: list[str]
    normalized_terms: list[NormalizedTerm]
    must_terms: list[str]
    should_terms: list[str]
    exclude_terms: list[str]
    filters: dict
    needs_clarification: bool
```

---

## 模块 2：Rewrite Engine

职责：

* 从 structured intent 生成多种 rewrite

接口：

```python
class RewriteCandidate(BaseModel):
    query: str
    purpose: str
    weight: float

class RewriteOutput(BaseModel):
    keyword_rewrites: list[RewriteCandidate]
    semantic_rewrites: list[RewriteCandidate]
    filters: dict
```

建议拆成两个 builder：

* `keyword_rewrite_builder.py`
* `semantic_rewrite_builder.py`

---

## 模块 3：Planner

职责：

* 根据 query 类型、rewrite 结果、运行时预算生成执行计划

接口：

```python
class RetrievalPlan(BaseModel):
    mode: str
    keyword_requests: list[dict]
    semantic_requests: list[dict]
    fusion: dict
    rerank: dict
    fallback: dict
```

planner 输入不仅看理解结果，还看：

* query 长度
* latency budget
* backend 可用性
* 是否灰度实验
* 当前用户是不是高优先级流量

---

## 模块 4：Executor

职责：

* 按 planner 发请求
* 并发执行
* fusion
* rerank
* 记录日志

---

# 十、一个推荐的目录结构

```text
query_understanding/
├── schemas/
│   ├── understanding_v1.json
│   ├── rewrite_v1.json
│   └── retrieval_plan_v1.json
├── prompts/
│   ├── system_prompt_v1.txt
│   └── fewshots_v1.json
├── normalizers/
│   ├── ingredient_alias.json
│   ├── function_alias.json
│   ├── property_alias.json
│   └── dosage_form_alias.json
├── rewrite/
│   ├── keyword_builder.py
│   ├── semantic_builder.py
│   └── rules.py
├── planner/
│   ├── planner.py
│   ├── policies.py
│   └── fallback.py
├── executor/
│   ├── keyword_client.py
│   ├── semantic_client.py
│   ├── fusion.py
│   └── rerank.py
├── eval/
│   ├── understanding_eval.py
│   ├── retrieval_eval.py
│   └── gold_queries.jsonl
└── service/
    └── api.py
```

---

# 十一、v1 最关键的实现策略

## 1. 先 rule-based planner，不要一开始就 LLM planner

很多人会想：

> planner 也让 LLM 来决定不就行了？

不建议 v1 这么做。

v1 最稳的是：

* LLM 负责理解
* 规则负责 planning

原因：

* 更可控
* 更容易 debug
* 更容易做 A/B
* 更容易做 fallback
* 延迟更稳

等你积累了足够请求日志，再考虑：

* 学习式 planner
* bandit / policy optimization
* reinforcement-style routing

---

## 2. query rewrite 建议“一主两辅”

v1 很实用的做法：

### 主 rewrite

最平衡的一条

### precision rewrite

更严格，偏 keyword

### semantic rewrite

偏自然语言、偏 dense retrieval

这样 planner 可按场景选用，不必每次只赌一条。

---

## 3. 属性词不要太早变成 hard constraint

这是材料 / 配方搜索里最容易踩的坑。

比如：

* 清爽
* 不厚重
* 手感好
* 稳定性强
* 低刺激

这些词：

* 可以作为 semantic query 描述
* 可以作为 rerank signal
* 可以作为 should terms

但通常不适合一开始全塞进 keyword must。

---

# 十二、给你一个完整的请求级示例

用户输入：

> 我想找适合敏感肌、清爽不泛白的物理防晒乳

---

## Step 1：LLM understanding 输出

```json
{
  "intent": "lookup_formulation",
  "core_terms": ["物理防晒", "防晒乳", "敏感肌", "清爽", "不泛白"],
  "normalized_terms": [
    {"source": "物理防晒", "canonical": "physical sunscreen", "type": "function", "confidence": 0.95},
    {"source": "防晒乳", "canonical": "lotion", "type": "dosage_form", "confidence": 0.90},
    {"source": "敏感肌", "canonical": "sensitive skin", "type": "application", "confidence": 0.78},
    {"source": "清爽", "canonical": "non-greasy", "type": "property", "confidence": 0.82},
    {"source": "不泛白", "canonical": "low whitening", "type": "property", "confidence": 0.87}
  ],
  "must_terms": ["physical sunscreen", "lotion"],
  "should_terms": ["sensitive skin", "non-greasy", "low whitening"],
  "filters": {
    "application": ["sun care"],
    "dosage_form": ["lotion", "emulsion"]
  }
}
```

---

## Step 2：rewrite 输出

```json
{
  "keyword_rewrites": [
    {
      "query": "\"physical sunscreen\" AND (lotion OR emulsion)",
      "purpose": "precision_core",
      "weight": 1.0
    },
    {
      "query": "\"physical sunscreen\" AND (lotion OR emulsion) AND (\"non-greasy\" OR \"low whitening\")",
      "purpose": "balanced",
      "weight": 0.9
    }
  ],
  "semantic_rewrites": [
    {
      "query": "physical sunscreen lotion formulation for sensitive skin with low whitening and non-greasy skin feel",
      "purpose": "dense_main",
      "weight": 1.0
    }
  ]
}
```

---

## Step 3：planner 输出

```json
{
  "mode": "hybrid_parallel",
  "keyword_requests": [
    {
      "query": "\"physical sunscreen\" AND (lotion OR emulsion)",
      "filters": {"application": ["sun care"]},
      "top_k": 100
    },
    {
      "query": "\"physical sunscreen\" AND (lotion OR emulsion) AND (\"non-greasy\" OR \"low whitening\")",
      "filters": {"application": ["sun care"]},
      "top_k": 50
    }
  ],
  "semantic_requests": [
    {
      "query": "physical sunscreen lotion formulation for sensitive skin with low whitening and non-greasy skin feel",
      "top_k": 100
    }
  ],
  "fusion": {"method": "rrf"},
  "rerank": {
    "enabled": true,
    "constraints": ["sensitive skin", "low whitening", "non-greasy"]
  },
  "fallback": {
    "on_zero_hit": "drop_soft_properties"
  }
}
```

这就是一个完整的工程闭环。

---

# 十三、如何评估 rewrite / planner 做得好不好

你不能只看“rewrite 看起来像不像人写的”。

真正该看的，是它有没有提升 retrieval。

---

## 先评估 rewrite

### 指标

* keyword recall@k 是否提升
* semantic recall@k 是否提升
* hybrid recall@k 是否提升
* rewrite 后是否出现过度扩展

### 常见坏例子

* 把 property 误扩展成 ingredient
* 把模糊诉求改写得过死
* semantic rewrite 太长，反而稀释主题

---

## 再评估 planner

### 指标

* 0-hit rate 是否下降
* hybrid 相比 single-path 是否提升
* strict -> relaxed fallback 是否有效
* latency 是否可接受
* rerank 前后 nDCG 是否提升

Evals 的核心思想就是：把这类模块当作结构化测试对象，持续测而不是凭感觉改。([developers.openai.com][2])

---

# 十四、你现在最值得做的 v1

如果我是你，我会这样落地：

## Phase 1：先把 planner 做成规则引擎

输入：

* structured understanding
* normalized terms
* query meta

输出：

* retrieval plan

## Phase 2：rewrite 只做三类

* keyword precision rewrite
* keyword balanced rewrite
* semantic dense rewrite

## Phase 3：加 3 个 fallback

* LLM fail -> raw query
* keyword 0-hit -> relax property constraints
* semantic too broad -> keep only core entities

## Phase 4：做离线集

每条 query 同时记录：

* raw query baseline
* rewrite 后结果
* planner 后结果

这样你能非常快看出：

* 是 LLM 理解有问题
* 是 rewrite 有问题
* 还是 planner 有问题

---

# 十五、最后给一个最简定义，方便你记

你可以记成：

### Query Rewrite

把用户 query 改写成适合不同检索器的查询表达。

### Retrieval Planner

根据 query 类型、结构化理解结果和系统能力，决定使用哪些检索器、如何分配约束、如何设置参数、如何融合结果的策略层。

---

如果你要，我下一条可以直接继续给你：

**一版“materials / formulation search 的 retrieval planner v1 伪代码 + Python 类设计”**，我会直接写到能落成服务骨架的程度。

[1]: https://developers.openai.com/api/docs/guides/structured-outputs/?utm_source=chatgpt.com "Structured model outputs | OpenAI API"
[2]: https://developers.openai.com/cookbook/examples/evaluation/use-cases/structured-outputs-evaluation/?utm_source=chatgpt.com "Evals API Use-case - Structured Outputs Evaluation"
