---
name: co-review
description: |
  所有检视/检查（代码）相关请求，必须首先判断模式。

  必须首先通过 `ask_user` 工具询问用户选择模式：

  - 普通 Review
  - Co-Review 模式
  
  如果用户选择“普通 Review”，则不启用本 Skill，由普通代码检视流程处理。
  
  只有当用户明确选择“Co-Review 模式”时，才启用本 Skill，并进入协同检视流程，
  
  Co-Review 协同代码检视 Skill。
  
  用于在代码检视过程中提供交互式反馈闭环、企业经验沉淀和检查能力演进能力。将一次性代码 Review 转化为可反馈、可修正、可演进的协同检视流程。

  支持：
  - 交互式反馈；
  - 检查结果修正；
  - 企业经验沉淀；
  - 检查能力持续演进。

  本 Skill 不负责普通代码 Review，不替代基础检查流程，仅在用户选择 Co-Review 模式后参与。
---

# Co Review Skill

## Overview

核心流程：

```text
用户发起检视
    ↓
识别检查目标
    ↓
执行检查
    ↓
展示结果
    ↓
用户反馈
    ↓
重新规划 / 重新检查
    ↓
用户确认
    ↓
知识沉淀
```

---



# Available Agents

本 Skill 只能调度以下 Agent：

| Agent                | Responsibility |
| -------------------- | -------------- |
| inspection-target    | 根据代码变化识别检查目标   |
| inspection-execution | 根据检查目标执行具体检查   |
| knowledge-ingestion  | 将确认后的经验沉淀为知识   |

不得绕过上述 Agent 自行完成专业任务。

---

# Responsibilities

## Skill Should

Co-Review Skill 负责协同检视流程编排和反馈闭环管理，不直接执行代码分析，而是根据用户意图和检视状态协调下游 Agent。

负责：

* 管理 Co-Review 检视流程，包括检查启动、反馈处理、重新检查和知识沉淀；
* 调度 `inspection-target` Agent 识别需要关注的检查目标；
* 基于 `inspection-target` 输出的 `InspectionTargetSet`，规划并组织 `inspection-execution` Agent 的检查任务；
* 根据当前检查上下文、目标信息和用户反馈，决定检查执行策略；
* 根据反馈影响范围选择合适的重新检查路径：
* 管理用户确认流程，确保检查结果经过用户认可后再进入知识沉淀阶段；
* 谨慎使用搜索功能，仅在必要时调用搜索以避免额外开销。
* 本功能只有inspection-target、inspection-execution、knowledge-ingestion3个Agent参与，不要调用其他Agent
* CoReview 必须严格且仅能基于实际执行的检查结果生成报告。
* Co-Review Skill 只负责流程编排和 Agent 协调，不承担具体分析和知识处理职责。

---

## Skill Should NOT


禁止：

* 自行分析漏洞或安全问题；
* 自行识别、创建、修改 `InspectionTarget`；
* 自行执行代码检查；
* 自行生成、修改或裁决 `InspectionFinding`；
* 自行判断代码是否安全；
* 自行抽取、验证或结构化用户知识；
* 在用户未明确确认或未授权时触发知识沉淀；
* 在没有对应 `InspectionExecution` 任务时，自行分析代码；
* 在 `Skip Inspection` 后继续补充其他风险；
* 自行发现新的检查点；
* 自行增加未经过 `InspectionTarget` 识别和路由的检查项。


---

# User Intent Routing

收到用户输入后，首先判断意图。

## Intent 1: Review Request

用户要求进行代码检查。

流程：

```text
User Request

↓

check

↓

Present Result

↓

Ask Feedback
```

检查完成后必须询问：

```
以上是本次检视结果。
你认为是否存在误报、漏报、判断不准确或需要补充的业务背景或者信息？
如果有，我可以根据你的反馈重新检查。
```

---

## Intent 2: Review Feedback

用户针对检查结果反馈：
流程：

```text
User Feedback

↓

FeedbackRecord

↓

Re-check

↓

Show Difference

↓

Confirm after recheck

↓

Knowledge Ingestion （用户确认后）
```

禁止：

```
收到反馈
    ↓
直接修改结果
```

说明：用户可能连续多轮补充信息，因此所有反馈必须先通过'FeedbackRecord'记录，再最后根据用于意愿统一通过Knowledge Ingestion处理，完成知识沉淀。

---

## 具体流程说明

如下内容为对'User Intent Routing'的详细说明，执行时需要严格遵守。

### check（re-check）

此流程为co-review最重要的流程，检查都应遵循此流程规定的内容执行。

当需要检查代码或者重新检查代码时，执行：

```text
Decide

↓

inspection-target

↓

Plan Inspection Execution

↓

inspection-execution

↓

Collect Results

↓

Present Updated Result
```

---

#### Decide

`Decide`在用户对上次检查有反馈的情况下生效，否则跳过此环节。

`Decide` 用于判断用户反馈的影响范围，是否影响当前检查目标还是只影响检查内容。

* 如果反馈可能导致检查目标变化（新增风险点、补充检查范围、发现遗漏场景等），重新调用 `inspection-target` Agent 识别和调整检查目标；
* 如果反馈仅影响已有检查目标的检查执行过程（补充业务语义、提供检查信息、纠正已有 Finding 等），保持当前 `InspectionTargetSet`，直接重新规划 `inspection-execution` Agent 的检查任务；



#### inspection-target

调用Agent：
```
inspection-target
```

##### 输入

包含如下内容：

```
用户想要检视的代码（必须）
用户生成代码的所用到的SPEC（可选）
用户反馈的信息（如果存在的情况）
```

##### 输出

InspectionTargetSet

```json
{
  "inspection_targets": [
    {
      "corncern_classification": "",
      "concrete_inspect": "",
      "related_knowledge": []
    }
  ]
}
```

说明：
corncern_classification:检查点的分类
concrete_inspect：关注的具体的检查内容
related_knowledge：和该检查项有关的知识

#### Plan Inspection Execution

遵照 'references/plan-inspection-exection.md'执行

#### inspection-execution

根据前序步骤输出的调动计划，调用Agent：
```
inspection-execution
```

##### 输入

每个 Agent 输入：

```text
ability(加载的检查能力)

+

Single InspectionTarget

+

相关用户 Feedback 信息
```

Feedback 大概分为两类：关于检查能力的反馈（如何检查）和关于检查知识的反馈（检查时会用的关键信息，如接口含义、业务知识等）

##### 输出

检查结果


### FeedbackRecord

接收到用户反馈后（仅针对用户反馈）：

须单独保存到一个记忆单元中，此单元不落盘，只存在本次执行的记忆或上下文中，供后续修改和使用。

该单元可以命名为'FeedbackRecord'，格式如下

```
{
  "original_feedback": [
    "用户原始反馈内容1",
    "用户原始反馈内容2"
  ]
}
```

要求：
* original_feedback 保留用户原文；
* 采用追加方式；
* 不覆盖历史反馈。
* 每次用户进行反馈均进行记录

### Knowledge Ingestion


只有用户明确同意沉淀后：

调用Agent：

```text
knowledge-ingestion
```

输入：

```text
FeedbackRecord

+

Original Inspection Result

+

Updated Inspection Result
```

由：

```text
knowledge-ingestion
```

负责：

* 反馈分析；
* 冲突处理；
* 知识提取；
* 知识验证；
* 知识结构化；
* 入库。

---


# 结果呈现方式

CoReview 在检查完成后，根据是否存在用户反馈采用不同交互方式。

## 首次检查

首次检查完成后：

- 仅展示最终检查结果；
- 不展示内部信息（如 InspectionTarget、Ability、Knowledge Retrieval、执行计划）。

如果发现问题，展示：
- 问题描述；
- 相关证据。

如果未发现问题，直接说明未发现问题。

然后询问用户（不要使用工具`ask_user`）：

```text
以上是本次检视结果。
如果存在误报、漏报，或者需要补充业务背景和规则信息，可以继续反馈，我会根据反馈重新检查。
```

---

## 反馈后的重新检查

基于用户反馈完成重新检查后：

* 展示最新检查结果；
* 不展示内部检查过程和调度信息。

然后询问用户（不要使用工具`ask_user`）：

```text
以上是根据反馈重新检视后的结果。
如果还有补充信息，可以继续反馈。
如果本轮反馈具有复用价值，是否需要沉淀为后续检视知识？
```

---

# 输出原则

* 仅展示最终检查结果：是否存在问题、问题是什么以及必要的证据信息，

* 不展示任何内部逻辑、流程信息。；
* 不展示 InspectionTarget、Ability、Knowledge 等内部执行信息；
* 不展示检查过程；
* 不输出无问题的检查列表；
* 无问题时直接说明未发现问题。
* 不要解释路由原则、规则和过程，例如:'..根据路由规则..无匹配检查能力、无相关检查知识、无用户补充信息，...不生成检查执行计划。'
* **严禁** 在最终输出展示中间过程决策信息、路由规则等，不要暴露任何内部决策过程

包括不限于：

- InspectionTarget；
- Ability Matching 结果；
- Knowledge Retrieval 结果；
- Routing Decision；
- Skip Inspection 原因；
- Skill 是否存在；
- Agent 调度过程；
- 内部执行计划 `InspectionExecutionPlan`

当前代码自演进能力未识别到可执行的检查项时：

- 严禁展示内部检查目标；
- 严禁展示 Ability 匹配情况；
- 严禁展示知识检索结果；
- 严禁展示路由决策过程。
- 严禁展示中间决策过程。

仅向用户说明：

```text
当前代码自演进能力未识别到需要进一步检查的风险项。
```