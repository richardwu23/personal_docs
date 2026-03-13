## 基于以上思路， 如果已经有了 keywords 搜索 和 语义搜索的接口。 从零开始构建一个基于LLM 的 用户的 query understanding.  应该怎么做，假设基于以上的 输出的prompt ,且已经有了 LLM api.  还需要做什么？  



可以把它理解成：**prompt + LLM API 只是 query understanding 的一个子模块**。
从零开始做一个可上线的 LLM query understanding，真正要补齐的是下面这整套工程链路：

OpenAI 官方现在明确支持用 **Structured Outputs** 让模型严格按你定义的 JSON Schema 返回结果，这比“自由文本抽关键词”稳定得多；官方 prompt 指南也强调，复杂任务要靠**明确指令、清晰输出格式、示例**来提升一致性；另外，官方也提供了针对 structured output 任务的 eval 思路，说明这类模块应该被当作**可评测的工程组件**，而不是只靠主观感觉调 prompt。([OpenAI Developers][1])

---

# 先讲结论

如果你已经有：

* keyword search 接口
* semantic / vector search 接口
* LLM API

那么下一步不是继续打磨一句 prompt，而是要补这 8 个东西：

1. **输出 schema**
2. **领域规范化体系**
3. **LLM query understanding service**
4. **后处理与校验**
5. **query rewrite / retrieval planner**
6. **离线评测集**
7. **线上日志与可观测性**
8. **兜底与降级策略**

没有这些，prompt 再漂亮也只是 demo，不是 production。

---

# 一、先定义模块边界

推荐把整个链路切成：

`User Query`
→ `Query Understanding (LLM)`
→ `Normalization / Validation`
→ `Retrieval Planning`
→ `Keyword Search + Semantic Search`
→ `Fusion / Rerank`
→ `Response`

其中你现在要做的是前面这段：

`User Query`
→ `LLM 理解`
→ `结构化结果`
→ `规范化`
→ `生成 keyword_query / semantic_query / filters`

也就是说，这个模块的职责不是“回答用户”，而是：

* 理解用户意图
* 识别实体 / 属性 / 约束
* 做领域规范化
* 把自然语言转换成搜索系统能吃的检索信号

---

# 二、第一件事：定 schema，而不是先调 prompt

建议第一版 schema 控制在**够用、可评测、可调试**的粒度。

例如：

```json
{
  "intent": "lookup_formulation|lookup_ingredient|lookup_property|troubleshooting|comparison|other",
  "language": "zh|en|mixed",
  "core_terms": ["string"],
  "normalized_terms": [
    {
      "source": "string",
      "canonical": "string",
      "type": "ingredient|function|property|dosage_form|application|process|material_class|other",
      "confidence": 0.0
    }
  ],
  "must_terms": ["string"],
  "should_terms": ["string"],
  "exclude_terms": ["string"],
  "filters": {
    "application": ["string"],
    "dosage_form": ["string"],
    "functional_role": ["string"],
    "property_target": ["string"],
    "material_class": ["string"]
  },
  "rewritten_queries": {
    "keyword_query": "string",
    "semantic_query": "string"
  },
  "needs_clarification": false,
  "clarification_reason": "string"
}
```

这个 schema 的意义是：

* 方便 LLM 稳定输出
* 方便程序做校验
* 方便做离线评测
* 方便你回放日志排查问题

这里最关键的不是字段多，而是字段**职责清楚**。

---

# 三、第二件事：做领域规范化层

这一步非常重要。
在材料/配方搜索里，**query understanding 的核心价值不只是抽词，而是规范化**。

你至少要准备下面几类知识：

## 1. 术语词表

例如：

* zinc oxide ↔ ZnO
* titanium dioxide ↔ TiO2
* polyvinylpyrrolidone ↔ PVP
* sodium hyaluronate ↔ hyaluronic acid（注意不总是完全等价）

## 2. 功能词表

例如：

* 保湿 → humectant / moisturization
* 成膜 → film-forming
* 乳化 → emulsifier / emulsification system
* 分散 → dispersant

## 3. 属性词表

例如：

* 清爽 → non-greasy / lightweight
* 不泛白 → low whitening / transparent appearance
* 防水 → water-resistant
* 稳定性好 → formulation stability

## 4. 剂型 / 应用场景词表

例如：

* lotion / cream / gel / spray
* sunscreen / hair care / coating / adhesive / polymer composite

这层不一定一开始就很大，但一定要有。
因为 LLM 可以帮你“猜”，但**生产系统的稳定性最终来自可控的 normalization layer**。

---

# 四、第三件事：把 LLM 调用包装成独立服务

不要把 prompt 直接散落在业务代码里。
最好做成一个独立服务，例如：

* `query-understanding-service`

输入：

```json
{
  "query": "我想找清爽不泛白的物理防晒乳配方",
  "user_context": {},
  "domain": "materials_formulation",
  "language_hint": "zh"
}
```

输出：

```json
{
  "intent": "lookup_formulation",
  "core_terms": ["物理防晒", "防晒乳", "清爽", "不泛白"],
  ...
}
```

服务内部至少包含：

* prompt template
* schema definition
* model config
* timeout / retry
* response validation
* post-processing
* logging

这样后面换模型、换 prompt、做 A/B test 才方便。

---

# 五、第四件事：不要让 LLM 直接产最终检索语句

更稳的做法是：

**LLM 先产中间表示**
→ **程序根据规则生成最终检索 query**

也就是：

* LLM 负责理解
* 程序负责组装

例如：

LLM 输出：

```json
{
  "must_terms": ["sunscreen", "lotion"],
  "should_terms": ["zinc oxide", "titanium dioxide", "transparent", "non-greasy"],
  "filters": {
    "application": ["sun care"],
    "dosage_form": ["lotion", "emulsion"]
  }
}
```

然后你自己的 retrieval planner 去生成：

### 给 keyword search 的 query

例如 BM25 / ES DSL：

```text
(application:sun care) AND (dosage_form:lotion OR emulsion) AND (sunscreen) AND (transparent OR non-greasy OR zinc oxide OR titanium dioxide)
```

### 给 semantic search 的 query

例如：

```text
physical sunscreen lotion formulation with inorganic UV filters, low whitening, transparent appearance, non-greasy skin feel
```

这样做的好处是：

* 可解释
* 可控
* 方便回放
* 方便逐步优化每一层

---

# 六、第五件事：加一个“后处理与校验”层

这一层经常被忽略，但上线非常关键。

你至少要做：

## 1. JSON schema 校验

如果结果不合法：

* 重试一次
* 或走 fallback

Structured Outputs 本身就是为此设计的，可以减少缺字段、错枚举这类问题。([OpenAI Developers][1])

## 2. 术语合法性校验

例如：

* 是否映射到了你领域词表中不存在的 canonical term
* 是否把“剂型”错放成“成分”
* 是否过度扩展到了不合理的 ingredient

## 3. 长度与噪声控制

例如：

* `should_terms` 不超过 8~12 个
* 删除重复项
* 过泛词降权，如 “good”, “effective”, “better”

## 4. 置信度裁剪

例如：

* `confidence < 0.6` 的不进入 must
* 低置信扩展只进入 semantic query，不进 keyword must clause

---

# 七、第六件事：做 retrieval planner

这是很多系统里最有价值、也最容易被低估的一层。

planner 的职责是决定：

* keyword 和 semantic 是否都查
* 哪些 terms 进入 must
* 哪些只做 should expansion
* filters 怎么下推
* 哪些约束只用于 rerank，不用于召回
* 何时降级

一个实用的第一版策略是：

## 情况 A：强约束查询

例如：

* 指定 ingredient
* 指定剂型
* 指定性能要求

策略：

* keyword + semantic 并发查
* must_terms 更严格
* filters 强下推

## 情况 B：模糊探索查询

例如：

* “找清爽一点的防晒体系”
* “找适合敏感肌的成膜方案”

策略：

* semantic 权重大一些
* keyword 只保留核心短语
* 少下硬 filters

## 情况 C：明显不完整

例如：

* “帮我找一种更好的体系”
* “透明、防水、便宜”

策略：

* 标记 `needs_clarification=true`
* 但先做 best-effort 检索，不要完全卡死

---

# 八、第七件事：离线评测集一定要先做

这个模块很容易“看起来挺聪明”，但实际拉垮召回。

所以必须先做一个小型 gold set。

## 至少准备 100~300 条 query

按类型分层：

* ingredient lookup
* formulation lookup
* property-driven search
* troubleshooting
* comparative query
* vague / underspecified query
* 中英混合 query
* 带商品名 / 缩写 / 俗称的 query

## 每条样本标注这些东西

* intent
* core entities
* normalized terms
* must / should / exclude
* filters
* 期望检索到的文档或 doc ids
* 至少 top-k 是否命中

## 评测指标

分两层：

### 1. understanding 层

* schema valid rate
* intent accuracy
* entity extraction precision / recall
* normalization accuracy
* must/should classification accuracy

### 2. retrieval 层

* Recall@k
* MRR / nDCG@k
* 有无比 baseline keyword-only 提升
* 有无比 semantic-only 提升

官方 prompt / structured output 指南强调结构化约束和示例能提升稳定性，而 eval cookbook 说明 structured output 任务本身就应该拿数据集去评；这正适合你的场景。([OpenAI Developers][1])

---

# 九、第八件事：先做一个能跑的 v1，别一开始多 agent

你的场景第一版推荐：

## v1 架构

* 单次 LLM 调用
* 严格 JSON schema
* 小型领域词表
* 简单 normalization
* 简单 planner
* keyword + semantic 并发
* 后面再做 fusion / rerank

不要一开始就：

* 多轮 self-reflection
* 多 agent decomposition
* query expansion agent + planner agent + verifier agent

那会让复杂度和延迟先爆掉。

---

# 十、线上必须要有的日志

每次请求至少落这些字段：

```json
{
  "raw_query": "...",
  "llm_model": "...",
  "prompt_version": "qu_v1",
  "schema_version": "1.0",
  "llm_output": {...},
  "normalized_output": {...},
  "keyword_query": "...",
  "semantic_query": "...",
  "search_result_doc_ids": [...],
  "latency_ms": 1234,
  "fallback_used": false
}
```

这样你后面才能回答这些关键问题：

* 是 LLM 理解错了，还是 search 接口召回差
* 是 normalization 带偏了，还是 rerank 没排好
* 哪类 query 最容易挂
* 哪个 prompt version 更好

---

# 十一、兜底与降级策略一定要有

线上不能假设 LLM 永远正常。

至少准备这几种 fallback：

## 1. LLM 超时 / 失败

直接退化为：

* 原 query 做 semantic search
* 原 query + 轻量规则分词做 keyword search

## 2. JSON 不合法

* 重试一次
* 失败则 fallback

## 3. 结果过度扩展

例如生成太多 ingredient

* 截断
* 只保留高置信 terms

## 4. 查询极短

例如用户只输：

* “防晒”
* “ZnO”
  这种场景可直接走轻量规则，甚至不必每次都打 LLM

---

# 十二、很重要：不是所有 query 都该走 LLM

推荐加一个 routing gate：

## 不需要 LLM 的 query

* 单个术语
* 明确 chemical name
* 明确 patent id / paper id
* 很短的精确查询

## 需要 LLM 的 query

* 自然语言描述
* 多约束组合
* 口语化需求
* 配方效果导向表达
* 中英混合 / 模糊表达

这样能降低：

* 成本
* 延迟
* 不稳定性

---

# 十三、一个务实的最小闭环

你现在从零开始，最合理的 MVP 是：

## 第 1 步

定义 schema + prompt + 3~5 个 few-shot

## 第 2 步

实现一个 `query_understand(query) -> structured_json`

## 第 3 步

加 normalization：

* 词表映射
* 去重
* 置信度裁剪

## 第 4 步

实现 planner：

* 生成 keyword_query
* 生成 semantic_query
* 生成 filters

## 第 5 步

接你现有的两个 search 接口，并发调用

## 第 6 步

做一个小型离线集，先比 baseline：

* 原 query 直接搜
* LLM 改写后搜

## 第 7 步

把日志打全，人工 review 失败样本

这就已经是一个完整的 v1 了。

---

# 十四、一个推荐的工程目录

```text
query_understanding/
├── prompts/
│   ├── system_prompt_v1.txt
│   └── fewshot_v1.json
├── schemas/
│   └── query_understanding_v1.json
├── normalizers/
│   ├── ingredient_lexicon.json
│   ├── property_lexicon.json
│   └── function_lexicon.json
├── planner/
│   ├── keyword_builder.py
│   └── semantic_builder.py
├── service/
│   └── query_understanding_service.py
├── eval/
│   ├── gold_queries.jsonl
│   └── eval_runner.py
└── logs/
```

---

# 十五、你最容易踩的坑

## 1. 把“抽关键词”当成目标

真正目标是**生成可检索的结构化中间表示**。

## 2. 让 LLM 直接拼 ES DSL

第一版不建议。先让程序拼。

## 3. 没有领域词表

只靠 LLM，漂移会很明显。

## 4. 没有评测集

最后只能靠感觉调 prompt。

## 5. 所有 query 都打 LLM

成本和延迟都不必要。

## 6. 没有 fallback

线上稳定性会很差。

---

# 十六、如果我是你，我会这样做 v1

一个非常实用的 v1 方案：

* **模型输入**：原始 query
* **模型输出**：严格 JSON
* **程序后处理**：

  * normalize
  * confidence trim
  * dedupe
* **planner**：

  * keyword_query
  * semantic_query
  * filters
* **检索执行**：

  * keyword / semantic 并发
* **评测**：

  * 先做 200 条 query gold set
* **上线策略**：

  * 10% 流量灰度
  * 对比 baseline recall / click / downstream success

---

# 十七、你可以把它理解成 4 个核心工件

最终你真正要产出的不是一个 prompt，而是这 4 个工件：

1. **Schema**
2. **Prompt + few-shot**
3. **Domain normalization lexicon**
4. **Eval dataset**

这 4 个凑齐，query understanding 才真正开始变成一个工程模块。

---

如果你愿意，我下一条可以直接给你一版**“材料/配方搜索系统 query understanding v1 技术方案”**，包括：

* 完整 JSON schema
* system prompt
* 5 个 few-shot 示例
* Python/FastAPI 版服务骨架
* planner 伪代码
* eval 数据集格式设计

[1]: https://developers.openai.com/api/docs/guides/structured-outputs/?utm_source=chatgpt.com "Structured model outputs | OpenAI API"
