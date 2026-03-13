


# 2. 如何做 alias 表

当前阶段的 alias 表，不是做百科式标准术语库，而是做一个 **corpus-aware 的最小 alias 表**：

> 基于当前专利语料中真实出现、对检索最有价值的表达，构建一版小而有效的映射表。

---

## 2.1 最小 alias 表优先做哪些类别

建议优先级如下：

### 第一优先级：核心 ingredient

最容易直接提升召回。

例如：

* `zinc oxide`
* `titanium dioxide`
* `avobenzone`
* `octocrylene`
* 常见 `silicone`
* 常见 `polymer`
* 常见 `surfactant`

### 第二优先级：常见属性词

用户 query 很多从属性出发。

例如：

* `transparent`
* `non-greasy`
* `water-resistant`
* `stable`
* `low whitening`
* `white cast`
* `adhesion`
* `spreadability`

### 第三优先级：常见功能词

例如：

* `UV filter`
* `emulsifier`
* `humectant`
* `film former`
* `dispersant`
* `thickener`

### 第四优先级：常见剂型词

例如：

* `lotion`
* `cream`
* `gel`
* `spray`
* `emulsion`
* `suspension`

---

## 2.2 alias 表的构建原则

alias 表不要完全自动生成，建议采用：

**候选生成 + 人工审核** 两阶段

原因是：

* 自动方法容易把“上下文相近但语义不同”的词误并到一起
* 检索场景里错误扩展会明显拉低 precision
* 第一版最重要的是稳，而不是全

---

## 2.3 候选生成怎么做

### 方法 1：基于 seed list 扩展

先圈出一小批高价值 canonical term，例如：

* `zinc oxide`
* `titanium dioxide`
* `transparent`
* `non-greasy`
* `lotion`

然后去语料中找它们的常见变体，例如：

* 缩写：`ZnO`
* 翻译：`氧化锌`
* 修饰写法：`micronized zinc oxide`
* 变体：`nano zinc oxide`

### 方法 2：基于上下文相似性找变体

抽取 seed term 的上下文窗口，寻找：

* 上下文高度相似
* 表达形式略有差异

例如：

* `zinc oxide`
* `ZnO`
* `zinc oxide particles`

### 方法 3：基于字符串规则做变体归并

对 ingredient 类尤其有效：

* 大小写统一
* 连字符归一
* 空格归一
* 单复数归一
* 化学式和英文名映射
* 常见缩写归一

### 方法 4：从高频 n-gram 中筛选候选

从语料中抽取 1～4 gram，筛选满足以下条件的词：

* 词频高
* 文档频率适中
* 出现在关键字段
* 上下文像 ingredient / property / function / dosage form

---

## 2.4 一定要做人工审核

自动生成只是候选，最后必须人工确认。

原因：

* 有些 alias 可以用于 keyword expansion
* 有些 alias 只适合 semantic hint
* 有些 alias 太宽泛，不能直接加入 query
* 有些 alias 虽然形式相关，但实际不是同义

---

## 2.5 alias 表建议的数据结构

### 1. `ingredient_alias`

字段建议：

* `canonical`
* `alias`
* `alias_type`

  * `abbr`
  * `translation`
  * `variant`
  * `broader`
  * `narrower`
* `confidence`
* `source`

  * `manual`
  * `corpus`
  * `llm_suggested`
* `active_flag`

### 2. `property_alias`

字段建议：

* `canonical_property`
* `alias`
* `polarity`
* `confidence`

### 3. `function_alias`

字段建议：

* `canonical_function`
* `alias`

### 4. `dosage_form_alias`

字段建议：

* `canonical_form`
* `alias`

---

## 2.6 alias 表内容长什么样

建议每个 canonical term 最终形成类似这样的结构：

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

这里要注意：

* 不是所有 alias 都应该无脑进 keyword query
* 有些 alias 适合做 `keyword_should`
* 有些更适合做 semantic side 的补充理解
* 太宽泛的词不能直接拿来扩展

---

## 2.7 alias 表的目标不要定太大

V1 不要试图覆盖整个领域。

第一版更实际的目标是：

* 先覆盖最影响检索效果的一小批高价值概念
* 先做 50～200 个 canonical concepts
* 每个 canonical concept 有 3～10 个高价值 alias
* 先验证对高频 query 的 recall 是否有显著提升

---

## 2.8 alias 表建设的实际落地流程

建议按下面顺序推进：

### 第一步：确定小范围高价值概念集

例如：

* 20～50 个核心 ingredient
* 20～50 个常见属性词
* 10～30 个功能词
* 10～20 个剂型词

### 第二步：从语料中生成候选 alias

来源包括：

* 高频变体
* 缩写
* 翻译
* 相似上下文词
* 高频 n-gram

### 第三步：人工审核与标注

确认：

* 是否同义
* 是否近义
* 是否过宽泛
* 适合 keyword 还是 semantic
* 是否 active

### 第四步：上线为小型 alias 表

先服务 query normalization / expansion，不要一开始就承担太多复杂职责。

### 第五步：基于 query log 和检索效果迭代

观察：

* 哪些 alias 提升了 recall
* 哪些 alias 导致误召回
* 哪些 concept 需要继续细化

---

如果你需要，我下一步可以继续把这份内容往下整理成更实操的版本，例如：

**“如何盘点数据 + 如何做 alias 表”的数据表 schema 设计清单**
或者直接给你一版 **可执行的 SQL / Python 脚本框架**。
