

## 现在 search 的索引结构是 基于patent 维度的doc的结构。 数据还没有做严格的normalization 也不清楚实际数据的情况。 基于以上情况 回答以下问题， 1.如何盘点数据， ingredients, formulation, process 的写法， 属性词 常见写法， application / dosage form 是否有稳定表达， 哪些字段最可用。  具体数据工程上如何做。  2.如何从语料里做最小的 alias 表。  先做一小批高价值词：

* 核心 ingredient
* 常见功能词
* 常见属性词。  数据工程方面具体怎么做。  假设已经圈出了一些范围。  
3.V1 阶段的 normalization 具体怎么做。 
4.再到更高的， 从用户的input  开始 整体流程 是否是以下这样？
`用户输入`
→ `LLM query understanding`
→ `规则/词表后处理`
→ `生成 keyword query`
→ `生成 semantic query`
→ `hybrid retrieval`
→ `rerank`* 常见剂型词
那么query understanding 具体 做什么？  prompt  +LLM  稳定输出一个结构？ 
规则/词表后处理  是否就是 normalization ？     这一步 输出 什么？ keywords query 和 semantic query  input ?   


对，你现在这个阶段，最重要的不是先把 prompt 写得多漂亮，而是先把**语料现状摸清楚**，再决定 v1 的 query understanding 和 normalization 怎么做。
你当前前提是：

* search index 是 **patent 维度 doc**
* 数据**没有严格 normalization**
* 甚至**不清楚真实数据长什么样**

在这种情况下，第一目标不是“做标准术语体系”，而是：

> 先把现有专利语料中的“可检索表达模式”盘点出来，
> 再做一个 **corpus-aware** 的 query understanding + normalization v1。

我按你的 4 个问题分别回答。

---

# 1. 如何盘点数据：ingredients / formulation / process / 属性词 / application / dosage form / 可用字段

这里本质上是在做一个 **语料审计（corpus audit）**。
目标不是先抽取得多准，而是先回答这几个现实问题：

1. 你的 patent doc 里到底有哪些字段
2. 哪些字段文本质量高
3. 哪些字段最适合 keyword search
4. 哪些字段最适合 semantic search
5. ingredient / formulation / process / property / application / dosage form 这些信息，通常出现在什么字段里、以什么写法出现

---

## 1.1 先做字段盘点，不要先做 NLP

先拉一份字段清单。
例如专利 doc 常见可能有：

* title
* abstract
* claims
* description
* independent_claims
* examples / embodiments
* assignee
* ipc / cpc
* publication_country
* language
* keywords（如果有）
* machine tags（如果有）

你要先做的是：

### 第一步：字段覆盖率统计

看每个字段：

* 非空率
* 平均长度
* P50 / P90 / P99 长度
* language 分布
* 去重后 unique ratio
* 是否存在大量模板化噪声

例如一张审计表：

| field       | non_null_rate | avg_len | p90_len | lang_mix_rate | comment           |
| ----------- | ------------: | ------: | ------: | ------------: | ----------------- |
| title       |          0.99 |      88 |     130 |             低 | 高价值               |
| abstract    |          0.95 |    1200 |    2500 |             中 | 高价值               |
| claims      |          0.90 |    8000 |   25000 |             中 | 高价值但噪声高           |
| description |          0.82 |   30000 |  100000 |             高 | 丰富但长且噪声高          |
| ipc/cpc     |          0.97 |       - |       - |             低 | 适合 filter，不适合文本召回 |

### 第二步：字段可读性抽样

从每个字段抽 100～300 条样本，人工看：

* 是否有 OCR 噪声
* 是否大量 boilerplate
* 是否有大量机器翻译痕迹
* ingredient / property / process 信息主要出现在什么位置

这一步不要省。
因为后面 query understanding 再聪明，也要依赖这些字段。

---

## 1.2 做“术语分布盘点”，不是直接做实体抽取

现在还不知道数据情况，所以先不要上来就搞复杂 NER。
先做 **term profiling**。

### 对 ingredients

做这些统计：

1. 候选术语频次
2. 大小写 / 缩写 / 连字符变体
3. 英文 / 中文 / 化学式混写情况
4. 共现关系

比如你可以先从这些 pattern 入手：

* 化学名 pattern
* INCI-like pattern
* 简写 pattern：ZnO, TiO2, PVP, PEG-100
* 含数字和连字符的成分名
* 常见后缀：oxide, dioxide, polymer, acrylate, glycol, silicone, surfactant 等

### 对 formulation

重点不是抽“配方”两个字，而是看常见表达框架：

* “A composition comprising…”
* “An emulsion containing…”
* “A formulation including…”
* “A cosmetic composition…”
* “A sunscreen composition…”

你要统计的是：

* composition / formulation / emulsion / dispersion / gel / cream / lotion / suspension 等词频
* 它们在 title / abstract / claims 中的分布
* 哪些是高频稳定表达，哪些只是偶发

### 对 process

先看工艺动词和工艺短语：

* mixing
* heating
* stirring
* emulsifying
* dissolving
* dispersing
* neutralizing
* homogenizing
* coating
* curing

统计：

* 高频 process verbs
* 常见 n-grams：under agitation, at elevated temperature, followed by, then added 等
* 它们主要分布在哪些字段

### 对属性词

这个最值得盘点，因为用户 query 往往就是这种词。

例如：

* transparent
* non-greasy
* water-resistant
* stable
* low whitening
* glossy / matte
* softness / adhesion / viscosity / spreadability

这里要做的不是先定义 taxonomy，而是先找：

* 高频属性词
* 属性词的同义写法
* positive / negative 对应写法
* 用户语言与专利语言的桥接可能

### 对 application / dosage form

看是否存在稳定表达：

* application: sunscreen, hair care, skin care, coating, adhesive, oral care
* dosage form: lotion, cream, gel, spray, stick, suspension, emulsion

统计这些词在 title / abstract / claims 的出现频次和共现稳定性。

---

## 1.3 数据工程上具体怎么做

你这个阶段最实用的是：**SQL + Python 双轨**

### A. 用 SQL / Spark / Trino 做基础画像

适合做：

* 字段覆盖率
* 长度分布
* 非空率
* 基础 token / n-gram 频率
* 指定词表命中率
* field-level sampling

例如思路：

1. 建一张 `patent_corpus_profile_sample`

   * doc_id
   * title
   * abstract
   * claims
   * description
   * language
   * cpc/ipc

2. 建一张 `field_stats`

   * field_name
   * non_null_cnt
   * avg_char_len
   * avg_token_len
   * p50/p90/p99

3. 建一张 `term_freq`

   * field_name
   * term
   * tf
   * df
   * sample_doc_ids

### B. 用 Python 做更灵活的文本分析

适合做：

* regex 候选抽取
* n-gram 统计
* collocation / PMI
* 术语变体归并
* 抽样人工 review 产物导出

建议最开始做几个 notebook / script：

* `profile_fields.py`
* `extract_candidate_terms.py`
* `build_ngram_stats.py`
* `sample_contexts.py`

---

## 1.4 具体建议的盘点产物

第一阶段你应该明确产出这几张表或文件：

### 1. field_profile

字段质量画像

### 2. term_candidates_ingredient

ingredient 候选词频表

### 3. term_candidates_property

属性词候选表

### 4. term_candidates_dosage_form

剂型词候选表

### 5. term_context_samples

每个候选词配 20～50 条上下文样本，供人工 review

### 6. field_recommendation

最后输出结论，例如：

* keyword search 优先字段：title + abstract + independent_claims
* semantic search 优先字段：abstract + description_chunk
* process 信息主要在 description / examples
* dosage form 在 title/abstract 中较稳定
* 属性词 title/abstract 可用，但 claims 中更噪

这个比直接做 NER 更重要。

---

# 2. 如何从语料里做最小 alias 表

你已经说了：先做一小批高价值词。这个方向是对的。

注意这里的 alias 表不是“百科标准词表”，而是：

> **当前专利语料里真实出现、且对检索有价值的表达映射表**

---

## 2.1 最小 alias 表先做哪几类

优先级建议：

### 第一优先级：核心 ingredient

因为最容易带来直接召回提升。

例如：

* zinc oxide
* titanium dioxide
* avobenzone
* octocrylene
* silicone 类
* 常见 polymer / surfactant / preservative 类

### 第二优先级：常见属性词

因为用户 query 很常从属性出发。

例如：

* transparent
* non-greasy
* water-resistant
* whitening / white cast
* stable
* adhesion
* spreadability

### 第三优先级：常见功能词

例如：

* UV filter
* emulsifier
* humectant
* film former
* dispersant
* thickener

### 第四优先级：常见剂型词

例如：

* lotion
* cream
* gel
* spray
* emulsion
* suspension

---

## 2.2 数据工程上怎么做 alias 表

这个最好是“候选生成 + 人工确认”两阶段，不要完全自动。

---

### 阶段 A：候选生成

#### 方法 1：基于 seed list 扩展

假设你已经圈定一些核心词，比如 `zinc oxide`

去语料中找它周围高频共现写法：

* ZnO
* zinc oxide particles
* micronized zinc oxide
* nano zinc oxide
* 氧化锌

可用方法：

* exact / fuzzy match
* case folding
* punctuation normalization
* regex variant generation
* embedding similarity（可做辅助，但别直接全信）

#### 方法 2：基于上下文相似聚类

对于某个 seed term，抽出上下文窗口，找上下文很相似但表面形式不同的候选词。

例如：

* zinc oxide
* ZnO
* zinc oxide particulate

#### 方法 3：基于字符串规则

对 ingredient 很有效：

* 大小写归一
* 去掉连字符差异
* 单复数归一
* 化学式与英文名双向关联
* 常见缩写归一

#### 方法 4：基于 n-gram 候选

从高频 1~4 gram 里筛：

* 词频高
* 文档频率适中
* 出现在关键字段
* 上下文像 ingredient/property/function/dosage form

---

### 阶段 B：人工审核

这一步一定要有。
最后 alias 表要长成这种结构：

```json
{
  "canonical": "zinc oxide",
  "type": "ingredient",
  "aliases": [
    "zinc oxide",
    "ZnO",
    "氧化锌",
    "micronized zinc oxide",
    "nano zinc oxide"
  ],
  "query_expansion_policy": {
    "keyword_should": ["zinc oxide", "ZnO", "氧化锌"],
    "semantic_hint": "inorganic UV filter used in sunscreen formulations"
  }
}
```

注意：
不是所有 alias 都应该无脑进 keyword query。
有些只适合 should，有些只适合 semantic hint。

---

## 2.3 alias 表怎么组织

建议最开始做成几张小表：

### ingredient_alias

* canonical
* alias
* alias_type（abbr / translation / variant / broader / narrower）
* confidence
* source（manual / corpus / llm_suggested）
* active_flag

### property_alias

* canonical_property
* alias
* polarity（positive / negative / neutral）
* confidence

### function_alias

* canonical_function
* alias

### dosage_form_alias

* canonical_form
* alias

---

## 2.4 最小 alias 表的目标

第一版不要追求覆盖全领域。
目标应该是：

* 先覆盖最常见、最影响召回的 50~200 个 canonical concepts
* 每个 canonical concept 先有 3~10 个高价值 alias
* 能够显著提升一批高频 query 的 recall

---

# 3. V1 阶段的 normalization 具体怎么做

你现在数据没严格规范化，所以 v1 normalization 要非常保守。
不要做成“把所有 query 强行翻译成标准术语”。

应该分成三层。

---

## 3.1 第一层：表面归一化

这是最基础的，不依赖领域知识。

包括：

* lowercasing（视语言而定）
* 全半角统一
* 连字符 / 空格 / 标点归一
* 去掉明显噪声字符
* 常见单位写法归一（如果 relevant）
* 简单词形归一（慎用）

输出还是原 query，只是 cleaner。

例如：

* `ZnO-based sunscreen lotion`
* `zno based sunscreen lotion`

归一后统一成较稳定形式，但保留原文。

---

## 3.2 第二层：检索导向 alias expansion

这是 v1 的核心。

例如用户输入：

“清爽不泛白的物理防晒乳”

经过 query understanding 后识别出：

* 物理防晒
* 防晒乳
* 清爽
* 不泛白

然后 normalization / alias expansion 变成：

* 物理防晒

  * canonical: inorganic UV filter
  * retrieval aliases: physical sunscreen, mineral sunscreen, zinc oxide, titanium dioxide, ZnO, TiO2
* 防晒乳

  * canonical: sunscreen lotion
  * retrieval aliases: sunscreen lotion, sunscreen emulsion, sun care lotion
* 清爽

  * canonical: non-greasy / lightweight skin feel
  * retrieval aliases: non-greasy, lightweight, low tack
* 不泛白

  * canonical: low whitening
  * retrieval aliases: low whitening, transparent, reduced white cast, white cast

注意这里不是替换，而是输出一个结构。

例如：

```json id="hgynji"
{
  "normalized_concepts": [
    {
      "source": "物理防晒",
      "canonical": "inorganic UV filter",
      "type": "function",
      "retrieval_aliases": ["physical sunscreen", "mineral sunscreen", "zinc oxide", "titanium dioxide", "ZnO", "TiO2"],
      "confidence": 0.86
    }
  ]
}
```

---

## 3.3 第三层：规则裁剪

不是所有扩展词都能直接放入 query。

所以要有规则：

* 高置信 alias 才进 keyword should
* 低置信 alias 只进 semantic query
* 过宽泛的 alias 不进 must
* 太具体的猜测不进 query

例如：

* `UV filter` 太泛，不进 keyword must
* `zinc oxide` / `titanium dioxide` 可以进 keyword should
* `inorganic UV filter` 更适合 semantic query

---

## 3.4 V1 normalization 的输出到底是什么

不是一个字符串。
而是一个**规范化后的中间表示**。

建议输出像这样：

```json id="kg1jl3"
{
  "cleaned_query": "清爽不泛白的物理防晒乳",
  "normalized_concepts": [
    {
      "source": "物理防晒",
      "canonical": "inorganic UV filter",
      "type": "function",
      "aliases_for_keyword": ["physical sunscreen", "mineral sunscreen", "zinc oxide", "titanium dioxide"],
      "aliases_for_semantic": ["inorganic UV filter", "physical sunscreen", "mineral sunscreen"]
    },
    {
      "source": "清爽",
      "canonical": "non-greasy skin feel",
      "type": "property",
      "aliases_for_keyword": ["non-greasy", "lightweight"],
      "aliases_for_semantic": ["non-greasy skin feel", "lightweight sensory profile"]
    }
  ],
  "must_concepts": ["sunscreen lotion"],
  "should_concepts": ["physical sunscreen", "zinc oxide", "titanium dioxide", "non-greasy", "low whitening"],
  "filters_candidate": {
    "application": ["sun care"],
    "dosage_form": ["lotion", "emulsion"]
  }
}
```

这个才是后面 keyword builder / semantic builder 的输入。

---

# 4. 整体流程是否是你写的那样？每一步具体做什么？

你写的这条链路是对的，而且很适合 v1：

`用户输入`
→ `LLM query understanding`
→ `规则/词表后处理`
→ `生成 keyword query`
→ `生成 semantic query`
→ `hybrid retrieval`
→ `rerank`

下面我把每一步说清楚。

---

## 4.1 用户输入

输入就是原始 query。

例如：

* “我想找清爽不泛白的物理防晒乳配方”
* “有没有含 ZnO 的透明防晒 composition”
* “找一种耐水的成膜体系”

这个阶段不做复杂处理，只做最基础的 text clean。

---

## 4.2 LLM query understanding 做什么？

对，**它的核心就是 prompt + LLM，稳定输出一个结构化结果**。
它不是直接生成最终 keyword query。

它负责的是：

### 1. 识别 intent

例如：

* lookup_formulation
* lookup_ingredient
* property_driven_search
* troubleshooting
* comparison

### 2. 抽取 query concepts

例如：

* ingredients
* function
* property
* dosage form
* application
* process

### 3. 判断约束关系

例如：

* 哪些是 must
* 哪些是 should
* 哪些是 exclude
* 是否存在 filters 候选

### 4. 给出保守的 canonical understanding

例如：

* “物理防晒” ≈ inorganic/mineral UV filter
* “清爽” ≈ non-greasy / lightweight
* “不泛白” ≈ low whitening / reduced white cast

---

### LLM query understanding 的输出

建议就是一个严格 schema，比如：

```json id="0nqv12"
{
  "intent": "lookup_formulation",
  "language": "zh",
  "raw_terms": ["物理防晒", "防晒乳", "清爽", "不泛白"],
  "concepts": [
    {
      "text": "物理防晒",
      "type": "function",
      "canonical_hint": "inorganic UV filter",
      "confidence": 0.85
    },
    {
      "text": "防晒乳",
      "type": "dosage_form",
      "canonical_hint": "sunscreen lotion",
      "confidence": 0.95
    },
    {
      "text": "清爽",
      "type": "property",
      "canonical_hint": "non-greasy/lightweight",
      "confidence": 0.78
    },
    {
      "text": "不泛白",
      "type": "property",
      "canonical_hint": "low whitening",
      "confidence": 0.83
    }
  ],
  "must_terms": ["防晒乳"],
  "should_terms": ["物理防晒", "清爽", "不泛白"],
  "exclude_terms": [],
  "filter_candidates": {
    "application": ["sun care"],
    "dosage_form": ["lotion"]
  }
}
```

---

## 4.3 规则/词表后处理 是否就是 normalization？

对，**可以认为这一层就是 normalization + validation + retrieval adaptation**。
但比单纯“normalization”稍微宽一点。

它主要做 4 件事：

### 1. 校验

* schema 合法性
* 字段缺失补默认值
* 置信度裁剪
* 去重

### 2. alias lookup

把 LLM 给出的 concept 映射到你自己的 alias 表

例如：

* `canonical_hint = inorganic UV filter`
* 查 alias 表得到：

  * physical sunscreen
  * mineral sunscreen
  * zinc oxide
  * titanium dioxide
  * ZnO
  * TiO2

### 3. retrieval-aware normalization

根据当前索引现实，决定：

* 哪些 alias 适合 keyword
* 哪些更适合 semantic
* 哪些过宽泛要丢掉
* 哪些应该保留原词

### 4. 输出标准中间表示

这个输出不是最终搜索结果，而是后续 query builder 的输入。

---

### 这一步输出什么？

这一步应该输出一个 **normalized retrieval plan input**。

比如：

```json id="s93b1h"
{
  "intent": "lookup_formulation",
  "normalized_query_repr": {
    "must_terms_keyword": ["sunscreen lotion"],
    "should_terms_keyword": [
      "physical sunscreen",
      "mineral sunscreen",
      "zinc oxide",
      "titanium dioxide",
      "non-greasy",
      "transparent",
      "low whitening",
      "white cast"
    ],
    "semantic_concepts": [
      "physical sunscreen lotion formulation",
      "inorganic UV filter",
      "non-greasy skin feel",
      "low whitening / reduced white cast"
    ],
    "filter_candidates": {
      "application": ["sun care"],
      "dosage_form": ["lotion", "emulsion"]
    }
  }
}
```

这个就是 `keyword query builder` 和 `semantic query builder` 的输入。

---

## 4.4 生成 keyword query 做什么？

这里不是 LLM 的职责，而是程序逻辑。

输入：上一步 normalized representation
输出：适合你当前 keyword search 接口的 query

例如可能生成：

* 布尔查询字符串
* ES DSL
* Solr query
* Vespa YQL
* 内部 search API 的 structured payload

例如：

```json id="ahk5yh"
{
  "query_type": "keyword",
  "must": ["sunscreen lotion"],
  "should": [
    "physical sunscreen",
    "mineral sunscreen",
    "zinc oxide",
    "titanium dioxide",
    "non-greasy",
    "transparent",
    "low whitening",
    "white cast"
  ],
  "filters": {
    "application": ["sun care"],
    "dosage_form": ["lotion", "emulsion"]
  }
}
```

注意：
如果当前 index 还没有结构化 filter 字段，那这里的 `filters` 只能先弱化，或者转成文本 should/must，不要假装它真能 filter。

---

## 4.5 生成 semantic query 做什么？

也是程序逻辑，但可能会比 keyword 更自由一点。

输入：normalized representation
输出：一条更适合 embedding / semantic search 的自然语言或短文本 query

例如：

`physical sunscreen lotion formulation with inorganic UV filters, low whitening, reduced white cast, and non-greasy skin feel`

或者中文 / bilingual 版本，取决于你的 embedding 体系。

这一步通常会：

* 保留核心 intent
* 用 canonical concepts 组织一句简洁查询
* 避免堆太多 alias

---

## 4.6 hybrid retrieval

这一步就是：

* keyword search
* semantic search

并发跑。
输出两路候选 doc。

如果现在 index 是 patent 维度 doc，那么召回的还是 patent docs，而不是 chunk。

---

## 4.7 rerank

最后对 hybrid 召回结果 rerank。
这个时候 rerank 用到的输入，往往也会用到 query understanding 结果，比如：

* intent
* must concepts
* property constraints
* formulation-related concepts

所以 query understanding 不只是服务 keyword/semantic 两路，也会服务 rerank。

---

# 最后，把你的问题浓缩成一句工程定义

你现在这条链路里：

## Query understanding 负责什么？

负责把用户自然语言 query 变成一个**结构化、可解释的中间语义表示**。

## 规则/词表后处理负责什么？

负责把这个中间表示，结合你的真实语料和 alias 表，变成**适配当前检索系统的 normalized retrieval representation**。

## normalization 是哪一步？

主要在“规则/词表后处理”这一步，但也可以认为它横跨：

* LLM 给出 canonical hint
* 规则层做 retrieval-aware normalization

## 这一步输出什么？

输出给后续两个 builder 用的中间表示：

* keyword query builder input
* semantic query builder input

而不是直接输出最终文档结果。

---

# 一个最务实的 v1 形态

如果我是你，我会这样切：

### LLM query understanding 输出

“我理解用户在找什么”

### normalization / post-process 输出

“针对当前语料和索引，我应该拿哪些词去搜”

### keyword builder 输出

“给 keyword API 的参数”

### semantic builder 输出

“给 semantic API 的参数”

这四层一分开，后面就很好调。

---

如果你要，我下一条我可以直接继续给你一版非常实操的内容：

**“patent 维度 doc 场景下，query understanding + normalization v1 的 JSON schema 设计”**
我可以把每一步的输入输出字段都给你列出来。
