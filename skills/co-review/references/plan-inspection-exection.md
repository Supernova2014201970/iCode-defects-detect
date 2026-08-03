# Plan Inspection Execution

根据：

- `InspectionTargetSet`
- 用户反馈信息

为每个 `InspectionTarget` 规划对应的 `inspection-execution` Agent 执行任务。

该阶段不执行具体检查，仅负责生成内部执行计划 `InspectionExecutionPlan`，用于后续调度 `inspection-execution` Agent。


---

# Task Granularity

每个 `InspectionTarget` 独立生成一个执行任务：

```text
InspectionTarget

        ↓

Inspection Execution Task
```

一个任务对应一个检查目标。

---

# Parallel Execution

支持多个 `inspection-execution` Agent 并行执行：

* 单次最多启动 10 个 sub-agent；
* 超出并发限制的任务进入等待队列；
* 不允许丢弃任何检查任务。

---

# Inspection Execution Routing

对于每个 `InspectionTarget`，按照以下流程决定执行方式：

```text
InspectionTarget

        ↓

Ability Matching

        ↓

Routing Decision

        ↓

Generate InspectionExecutionPlan
```

---

# Ability Matching

## Matching Rule

根据：

```text
InspectionTarget.concern_classification
```

在：

```text
Linux：`$HOME/chrys/co-review/ability/skills`
windows：`%APPDATA%\chrys\co-review\ability\skills`
```

目录下按照 Skill 名称进行匹配。


匹配规则：

```text
InspectionTarget.concern_classification

            ==

Skill Name
```

即：

* 如果存在名称与 `concern_classification` 完全一致的 Skill，则认为匹配成功；
* 如果不存在对应 Skill，则认为无匹配 Ability。

匹配结果：

```text
Matched:

ability = concern_classification


Not Matched:

ability = null
```

---

## Routing Decision

根据：

* `ability`
* `InspectionTarget.related_knowledge`
* `user_feedback_context`

共同决定是否执行检查。

决策规则：

```text
InspectionTarget

        ↓

Ability Matching

        ↓

Check Execution Context

        ↓

Generate InspectionExecutionPlan
```

---

# Routing Rules

## Case 1: Has Matching Ability

条件：

```text
ability != null
```

决策：

```text
Execute Inspection
```

执行任务：

```yaml
ability:
  <matched ability>
```

说明：

* 使用专用检查能力；
* `related_knowledge` 和 `user_feedback_context` 作为检查上下文补充。

---

## Case 2: No Matching Ability but Has Related Knowledge

条件：

```text
ability == null

AND

InspectionTarget.related_knowledge != empty
```

决策：

```text
Execute Inspection
```

执行任务：

```yaml
ability:
  null
```

说明：

* 当前不存在匹配的专用检查能力；
* 使用通用检查能力；
* 基于 `related_knowledge` 完成检查。

---

## Case 3: No Matching Ability and No Related Knowledge

条件：

```text
ability == null

AND

InspectionTarget.related_knowledge == empty

```

决策：

```text
Execute Inspection
```

执行任务：

```yaml
ability:
  null
```

说明：

* 当前没有专用检查能力；
* 没有已有检查知识；
* 若存在用户反馈，结合用户反馈执行检查，使用通用检查能力，
* 若不存在用户反馈，使用通用检查能力，

---

# InspectionExecutionPlan

完成所有 `InspectionTarget` 路由决策后，生成内部执行计划：

```text
InspectionExecutionPlan
```

该计划：

* 仅用于系统内部调度；
* 不展示给用户；
* 后续基于该计划启动 `inspection-execution` Agent。

结构：

```json
{
  "inspection_execution_plan": [
    {
      "batch_id": 1,
      "ability": "",
      "inspection_target": {},
      "user_feedback_context": {
        "ability_feedback": [
          "需要重点检查动态 SQL 拼接场景"
        ],
        "knowledge_feedback": [
          "SafeSQLBuilder 已完成参数化处理，不应报告 SQL Injection"
        ]
      }
    }
  ]
}
```

字段说明：

| 字段                      | 说明                        |
| ----------------------- | ------------------------- |
| `batch_id`              | 执行批次编号，受最大 10 并发限制控制      |
| `ability`               | 匹配到的检查能力；无匹配时为空，但仍可能基于知识或用户反馈执行检查           |
| `inspection_target`     | 当前执行任务对应的检查目标             |
| `user_feedback_context` | 当前检查目标相关的用户反馈，仅在存在有效反馈时生成 |

---

# User Feedback Context

`user_feedback_context` 为可选字段。

只有用户反馈包含与当前检查目标相关的信息时才生成。

用户反馈主要分为两类：

## 1. Ability Feedback

影响：

* 检查方式；
* 检查策略；
* 检查深度。

例如：

```text
需要重点检查动态 SQL 拼接场景
```

---

## 2. Knowledge Feedback

影响：

* 检查判断依据；
* 业务语义；
* 安全控制信息。

例如：

```text
SafeSQLBuilder 已完成参数化处理，不应报告 SQL Injection
```

无相关反馈时：

```json
"user_feedback_context": null
```

或不生成该字段。

---


# 输出限制

禁止输出**任何**匹配、路由判断、路由决策等中间信息给用户，用户无需感知中间过程，只需感知结果即可

禁止输出：

- Ability 是否匹配；
- Skill 是否存在；
- Knowledge 是否为空；
- Routing Rules 判断；
- Target 逐项决策；
- Skip Inspection 原因；
- ExecutionPlan 生成过程。

这些内容仅用于内部执行，不属于用户检视结果。

当没有可执行检查项时，直接输出：

```text
当前代码自演进能力未识别到需要进一步检查的风险项。

以上是本次检视结果。如果存在误报、漏报，或者需要补充业务背景和信息，可以继续反馈，我会根据反馈重新检查。
```

---