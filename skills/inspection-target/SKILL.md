---
name: inspection-target
description: |
  从代码变更中识别值得进一步检查的目标，
  基于代码信号、用户 Spec 和检查知识生成结构化 InspectionTargetSet。
---

# Inspection Target Skill

## Overview

`InspectionTarget Skill` 负责从代码变更中识别潜在检查目标。

该 Skill 不执行具体代码检查，而是：

- 从代码变化中提取结构化 Code Facts；
- 根据代码信号发现潜在 Concern；
- 从知识库中召回相关检查知识；
- 综合代码、Spec 和知识生成 InspectionTargetSet。

---

# Workflow

```text
Code Change (patch / diff) + 用户的反馈

        ↓

1. Extract Code Facts
   提取结构化代码信号

        ↓

2. Retrieve Related Knowledge
   基于代码锚点匹配检查知识

        ↓

3. Identify Concerns
   根据代码信号、Spec 用户的反馈、和知识识别检查关注点

        ↓

4. Generate InspectionTargetSet
   生成结构化检查目标
```

---

# Step 1: Extract Code Facts

## Purpose

将代码变更抽象为统一的代码信号（Code Facts），用于后续知识匹配和检查目标发现。

## Representation

每个代码信号由：

* `symbol`
* `kind`

两个字段组成。

例如：

代码：

针对
```java
cryptoutil.decryptv2(cipherText)
```
中的`decryptv2`

提取：

```json
{
  "symbol": "decryptv2",
  "kind": "call"
}
```

其中：

* `symbol` 表示代码元素名称；
* `kind` 表示代码元素类型。

---

## Kind Definition

`kind` 为枚举字段，用于限定代码节点类型。

当前支持：

| kind       | 描述        | 示例                   |
| ---------- | --------- | -------------------- |
| call       | 函数或方法调用   | `decryptv2()`        |
| field      | 字段或属性访问   | `user.password`      |
| type       | 类型使用      | `CryptoUtil`         |
| config     | 配置项       | `enable_crypto=true` |
| annotation | 注解或装饰器    | `@Override`          |
| literal    | 字符串或数值字面量 | `"admin"`            |

---

# Step 2: Retrieve Related Knowledge

## Purpose

根据 Code Facts 从 `knowledge_path` 中检索与当前代码相关的检查知识。


## Knowledge Source


知识存储路径：

`knowledge_path` :
```text
Linux：`$HOME/chrys/co-review/knowledge`
windows：`%APPDATA%\chrys\co-review\knowledge`
```


每个 JSON 文件包含多个 Knowledge Item。

---

## Anchor Matching

每个 Knowledge Item 包含：

```json
{
  "anchors": {
    "code_signals": {
      "core": [],
      "context": []
    }
  }
}
```

匹配分两个阶段：

---

## Phase 1: Core Signal Matching

使用：

```text
CodeFact

        ↓

Knowledge.anchor.code_signals.core
```

进行匹配。

匹配规则：

```text
symbol 相等

AND

kind 相等
```

例如：

Code Fact:

```json
{
  "symbol": "decryptv2",
  "kind": "call"
}
```

Knowledge Anchor:

```json
{
  "symbol": "decryptv2",
  "kind": "call"
}
```

则：

```text
Core Match = true
```

---

## Phase 2: Context Signal Matching

Core Match 成功后，进一步结合代码上下文验证。

例如：

Knowledge:

```json
{
  "core": [
    {
      "symbol": "decryptv2",
      "kind": "call"
    }
  ],
  "context": [
    {
      "symbol": "cryptoutil",
      "kind": "type"
    }
  ]
}
```

代码：

```java
cryptoutil.decryptv2(data)
```

满足：

```text
core:
decryptv2(call)

+

context:
cryptoutil(type)
```

则认为：

```text
Knowledge Match Success
```

否则：

```text
No Related Knowledge
```

---

## Retrieved Knowledge Output

只有完整匹配的 Knowledge Item 才能进入：

```text
InspectionTarget.related_knowledge
```

禁止：

* 仅 core 匹配就返回；
* 模型自行补充未检索知识；
* 使用未命中的 Knowledge。

---

# Step 3: Generate InspectionTarget

## Purpose

基于：

* Code Facts；
* 用户提供 Spec；
* Retrieved Knowledge；
* 用户反馈信息

生成最终检查目标。检查目标是基于代码特征、上下文信息和知识经验识别出的潜在风险点。仅当存在合理风险依据时，才生成对应检查目标，并交由后续检查流程验证。

生成过程：

```text
Code Facts

+

User Spec

+

Related Knowledge

+

User Feedback

        ↓

Inspection Target Generation

        ↓

InspectionTargetSet
```

---

# Output Format

输出给主Agent，用于进一步的检查，不落盘：

```json
{
  "inspection_targets": [
    {
      "concern_classification": "",
      "concrete_inspect": "",
      "related_knowledge": []
    }
  ]
}
```

---

# Field Definition

| 字段                     | 描述            | 来源                      |
| ---------------------- | ------------- | ----------------------- |
| concern_classification | 检查目标分类        | Concern Classification  |
| concrete_inspect       | 具体需要关注的检查内容   | Code + Spec + Knowledge |
| related_knowledge      | 与该检查目标相关的知识列表 | Knowledge Retrieval     |


---

## concern_classification

分类标准：

```text
references/classification.json
```

分类原则：

```text
references/Concern-Classification.md
```

要求：

* 必须使用规定分类；
* 不允许自行创建分类；
* 一个 InspectionTarget 对应一个主要 Concern。

## concrete_inspect

* concrete_inspect 必须严格匹配 concern_classification 的检查范围，仅描述该分类下需要关注的具体问题。禁止为了丰富描述而引入无关风险信息，避免一个检查目标混合多个不同类型的检查内容。


描述：

> 当前检查目标需要重点关注的具体代码行为。

来源：

* Code Facts；
* 用户 Spec；
* Related Knowledge。
* 用户反馈


要求：

* 描述具体检查对象；
* 避免泛化描述；
* 可以直接作为后续 InspectionExecutionAgent 的检查输入。

例如：

```text
检查 cryptoutil.decryptv2 调用是否使用不安全密钥管理方式，
以及解密后的敏感数据是否存在暴露风险。
```

---

## related_knowledge

表示：

与当前 `InspectionTarget` 相关的、经过 `Knowledge Retrieval` 匹配得到的检查知识。

规则：

- 必须来自 `Knowledge Retrieval` 结果；
- 未匹配成功的知识不得加入；
- 如果没有匹配知识，则为空数组；
- 该字段存储完整知识内容，而非知识 ID 或引用路径；
- 后续 `inspection-execution` Agent 可直接使用该字段进行检查，无需再次查询知识库，降低系统交互开销。


每条知识保留以下字段：

```json
{
  "summary": "",
  "applicable_scenarios": [],
  "status": "",
  "precision": "",
  "scope": "",
  "review_effect": ""
}

```
---

# Core Principles

## 1. Target Discovery Only

本 Skill 负责：

```text
Code

↓

InspectionTarget
```

不负责：

* 漏洞分析；
* 代码检查；
* Finding 生成。

---

## 2. Knowledge Must Be Retrieved

所有：

```text
related_knowledge
```

必须来自：`knowledge_path`检索结果。

禁止：

* 模型幻觉补充；
* 根据经验虚构知识。

---

## 3. Evidence Driven

InspectionTarget 必须能够追溯到：

```text
Code Facts

+

Knowledge

+

Spec

+

用户反馈
```

避免无依据生成检查目标。