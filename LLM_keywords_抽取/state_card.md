下面是按 [state_card.md](/Users/richardwu/richard_code/personal_docs/LLM_keywords_抽取/state_card.md) 结构整理的一版。

## 1. 目标

在 `patent 维度 doc`、数据尚未严格 normalization、已有 `keyword search` 和 `semantic search` 接口的前提下，构建一个 **面向材料/配方领域的 query understanding v1**。

目标不是单纯“抽关键词”，而是把用户输入转成一份 **保守、结构化、可执行、可评测** 的检索中间表示，供后续生成 keyword query、semantic query，并进入 hybrid retrieval 和 rerank。

## 2. 当前共识

- 主流程是：
  `用户输入 -> LLM query understanding -> 规则/词表后处理 -> keyword query -> semantic query -> hybrid retrieval -> rerank`
- 当前阶段最重要的不是调 prompt，而是先做 **corpus audit**，摸清真实语料怎么写。
- normalization 必须是 **corpus-aware / retrieval-aware**，不能脱离实际索引数据做“理想标准化”。
- alias 表第一版应是 **最小、高价值、人工审核** 的映射表，而不是大而全术语库。
- LLM 应输出 **结构化 JSON 中间表示**，不要直接生成最终 ES DSL / 检索语句。
- 后处理层不仅是 normalization，还包括：
  结构校验、alias 映射、语义校验、逻辑冲突检查、置信度裁剪、生成 planner 可消费的中间表示。
- planner 和 rewrite 是独立层：
  rewrite 负责“搜什么表达”，planner 负责“怎么搜、哪几路搜、must/should/filter 怎么分配”。

## 3. 关键待验证点

- patent doc 中哪些字段最适合 keyword search，哪些更适合 semantic search。
- ingredients / formulation / process / property / application / dosage form 在真实语料中的稳定表达方式是什么。
- 当前索引里是否存在足够稳定的 application / dosage form / material class 承载字段，可否下推 filter。
- 哪些 alias expansion 确实能提升 recall，哪些会带来误召回。
- 属性词在 keyword 召回中的价值有多大，是否更适合放到 semantic / rerank。
- LLM 输出 schema 在真实 query 上的稳定性如何，是否足够支撑 v1。
- 哪些 query 类型值得走 LLM，哪些应直接轻量规则或原 query 检索。

## 4. 已做决策

- v1 先走 **单次 LLM 调用 + 严格结构化输出 + 规则后处理 + rule-based planner**。
- LLM 侧采取 **保守 normalization**：
  命中 alias 表再规范化；不确定时保留原词，不脑补具体 canonical。
- 检索侧采用 **keyword + semantic 并发** 的 hybrid 方案，再做融合和 rerank。
- 先建设 **最小 alias 表**，优先覆盖：
  核心 ingredient、常见属性词、常见功能词、常见剂型词。
- 先做 **SQL / Spark / Trino + Python** 的语料盘点和候选词统计，不先做复杂 NER。
- 后处理输出应是“校验后中间表示”，供 planner 使用，而不是直接把原始 LLM 输出下游透传。
- 必须准备 fallback：
  LLM 超时、JSON 非法、query 极短、rewrite 过激时可退化为原 query 检索。

## 5. 下一步

1. 做字段级语料盘点  
   产出 `field_profile / field_recommendation / term_candidates / context_samples`。

2. 抽样人工 review  
   确认 ingredients、process、property、application、dosage form 的真实表达模式。

3. 做最小 alias 表 v1  
   先覆盖一小批高价值 canonical concepts，并区分哪些 alias 适合 keyword、哪些只适合 semantic。

4. 定义 query understanding schema  
   明确 `intent / entities / constraints / retrieval_plan / clarification` 的结构。

5. 实现后处理与校验层  
   包括 schema 校验、alias 映射、语义检查、逻辑检查、置信度裁剪。

6. 实现 rule-based retrieval planner v1  
   生成 keyword query、semantic query、filters、fallback 策略。

7. 建离线 eval 集  
   评估 understanding 准确率、normalization 效果、Recall@k、nDCG@k。

如果你要，我下一步可以继续直接给你一版“可粘贴到 `state_card.md` 的精简终稿”。