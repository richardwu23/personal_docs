

# 目标
实现 Query Understanding 


# 当前结论 


# 确认决策 

## 快速上线v0 





# 未解决问题

# next move



搜索的整体流程是：
`用户输入`
→ `LLM query understanding`
→ `规则/词表后处理`
→ `生成 keyword query`
→ `生成 semantic query`
→ `hybrid retrieval`
→ `rerank`




可以，但前提要改成：

**这不是“完整 v1”，而是一个“保守、低依赖、可快速上线的 v0”**。

因为你跳过了：
1. 语料盘点
2. 抽样 review
3. alias 表

那就意味着这版不能做太多“领域规范化”，否则很容易把 query 带偏。  
所以这版的核心原则应该是：

- **少做 normalization**
- **不脑补 canonical term**
- **不做激进 expansion**
- **LLM 只产结构化理解，不直接决定复杂检索策略**

## 1. 问题原因

如果现在直接上 LLM，而没有语料盘点和 alias 表，最大风险不是“抽不出来”，而是：

- 抽出来的词 **和索引里的真实表达对不上**
- 模型会做“概念正确但检索错误”的改写
- application / dosage form / property 这些字段，模型以为能 filter，实际上索引未必支持
- 错误扩展会明显拉低 precision

所以快速上线版必须把目标降下来：

> 先让 LLM 把用户 query 变成稳定的结构化中间表示，  
> 再用非常保守的规则把它转成 keyword query 和 semantic query。

## 2. 修改步骤

### 第一步：把目标限定成“query understanding v0”

这一版只做 4 件事：

- 识别 `intent`
- 抽取 query 中明确出现的 `entities / constraints`
- 生成一个简短的 `semantic_query`
- 给检索层提供一个非常保守的 `retrieval hints`

不要做：

- 大规模 canonical normalization
- 大规模 synonym expansion
- 复杂 filter 下推
- 自动推断具体 ingredient

---

### 第二步：定义一个最小 schema

建议先用很小的输出结构，不要一开始太复杂。

```json
{
  "query_text": "string",
  "query_language": "zh|en|mixed",
  "intent": {
    "primary": "formulation_lookup|ingredient_lookup|property_lookup|process_lookup|comparison|exploratory|other",
    "confidence": 0.0
  },
  "mentions": [
    {
      "text": "string",
      "type": "ingredient|property|dosage_form|application|process|other",
      "role": "core|constraint|exclude",
      "confidence": 0.0
    }
  ],
  "retrieval_hints": {
    "must_terms": ["string"],
    "should_terms": ["string"],
    "exclude_terms": ["string"],
    "semantic_query": "string"
  },
  "clarification": {
    "needed": false,
    "reason": ""
  }
}
```

这里故意不放复杂 `canonical` 字段。  
因为现在没有 alias 和 corpus evidence，硬做 canonical 很容易错。

---

### 第三步：prompt 设计走“保守抽取”

system prompt 要明确要求：

- 只抽取用户 query 中明确出现或强显式暗示的内容
- 不要发明 ingredient / property / dosage form
- 不要把 vague property 强行转成专业术语，除非非常确定
- `must_terms` 只放 query 里最核心的 1 到 4 个词
- `should_terms` 只放弱扩展，且数量受限
- `semantic_query` 用自然语言重写，但不能添加 query 中不存在的具体化学物

---

### 第四步：程序侧加一个最小后处理层

即使是快速版，也必须有后处理。

这层至少做：

- JSON schema 校验
- 去重
- 长度裁剪
- 低置信词降级
- 禁止某些类型直接进 hard filter
- 冲突检查

最简单规则可以是：

- `confidence < 0.6` 不进 `must_terms`
- property 默认不进 keyword must
- application / dosage form 先不做 hard filter，除非用户说得非常明确
- `must_terms` 最多 4 个
- `should_terms` 最多 6 个

---

### 第五步：query builder 不要太聪明

#### keyword query 生成原则

- `must_terms` 只用用户原词或 LLM 抽出的原始 mention
- `should_terms` 只做轻量补充
- 不做复杂 synonym expansion

例如用户 query：

`我想找清爽不泛白的物理防晒乳配方`

可以保守生成：

- keyword must:
  - `防晒乳`
- keyword should:
  - `物理防晒`
  - `清爽`
  - `不泛白`

不要直接扩到：

- `zinc oxide`
- `titanium dioxide`
- `inorganic UV filter`

除非 query 里自己写了。

#### semantic query 生成原则

semantic side 可以稍微自由一点，但仍要保守。  
例如：

`physical sunscreen lotion formulation with non-greasy feel and low whitening preference`

这里可以做轻微技术化改写，但不要引入具体 ingredients。

---

### 第六步：planner 用固定规则，不要让 LLM 决策太多

快速版 planner 建议直接写死几条规则：

- 短 query、实体 query：直接 keyword + raw semantic
- 自然语言 query：走 LLM understanding + hybrid
- property 驱动 query：semantic 权重更高
- 明确 ingredient query：keyword 权重更高

v0 最稳的默认策略：

- `mode = hybrid`
- keyword 和 semantic 并发
- 简单融合
- rerank 可选

---

### 第七步：加 routing gate

不是所有 query 都打 LLM。

建议：

**不走 LLM**
- 单个术语
- 明确 chemical name
- 明确 patent id
- 很短的 1 到 3 token query

**走 LLM**
- 自然语言长 query
- 多约束 query
- 属性驱动 query
- 中文口语化 query
- 中英混合 query

这样上线更稳，也更省成本。

---

### 第八步：日志和回放必须先有

你即使先不上语料盘点，也要先有日志，不然后面没法补课。

至少记录：

```json
{
  "raw_query": "...",
  "llm_output": {},
  "validated_output": {},
  "keyword_query": "...",
  "semantic_query": "...",
  "final_doc_ids": [],
  "latency_ms": 0,
  "fallback_used": false
}
```

因为你后面真正要补的，不只是 alias，而是看：

- LLM 哪类 query 理解错
- 哪些 should terms 在误召回
- 哪些 query 根本不该走 LLM
- 哪些 canonical 化冲动必须压住

---

### 第九步：准备 fallback

快速上线版最怕 LLM 不稳定，所以一定有 fallback：

- LLM 超时：直接原 query 做 semantic + keyword
- JSON 非法：重试一次，失败就 fallback
- 输出过长：裁剪
- must_terms 为空：退化为 raw query hybrid
- query 太短：绕过 LLM

---

### 第十步：上线策略要灰度

这版不要一上来全量替换。

建议：

- 先只对“自然语言复杂 query”启用
- 先做小流量灰度
- 先看 top-k 召回是否明显变差
- 先人工 review 一批日志，再继续放量

## 3. 需要改动的文件和代码片段

如果你现在只是方案推进，不一定要改现有文档；但从实现角度，最小需要这些模块。

### 需要的模块

- `query_understanding_service`
- `query_schema`
- `query_postprocessor`
- `keyword_query_builder`
- `semantic_query_builder`
- `retrieval_router`
- `fallback_handler`

### 一个可上线的最小流程

```text
User Query
-> Routing Gate
-> LLM Query Understanding
-> Schema Validation
-> Conservative Post-processing
-> Keyword Query Builder
-> Semantic Query Builder
-> Hybrid Retrieval
-> Simple Fusion
-> Rerank(optional)
```

### 一个最小伪代码

```python
def handle_query(query: str):
    if should_bypass_llm(query):
        return hybrid_search(
            keyword_query=build_keyword_from_raw(query),
            semantic_query=query,
        )

    llm_output = query_understanding_llm(query)
    parsed = validate_schema(llm_output)
    safe = postprocess_conservatively(parsed)

    keyword_query = build_keyword_query(
        must_terms=safe["retrieval_hints"]["must_terms"],
        should_terms=safe["retrieval_hints"]["should_terms"],
        exclude_terms=safe["retrieval_hints"]["exclude_terms"],
    )

    semantic_query = safe["retrieval_hints"]["semantic_query"] or query

    return hybrid_search(
        keyword_query=keyword_query,
        semantic_query=semantic_query,
    )
```

### 一句最实用的落地建议

如果先不做语料盘点、review、alias 表，那就把这版定义成：

> **“LLM 结构化拆 query + 保守 hybrid builder”**

而不要定义成：

> **“LLM normalization + 智能 query rewrite 系统”**

前者能快速上线，后者在当前信息不足的前提下很容易失控。

如果你愿意，我下一步可以继续直接给你一版：
**“快速上线版 query understanding v0 的 prompt + JSON schema + 后处理规则清单”**。