# Clarification

用于在知识或能力摄入过程中，向用户补充关键信息。

基本流程：

```text
VerificationGateResult
    ↓
ClarificationQuestionSet
    ↓
ClarificationQuestion
    ↓
Ask User
    ↓
Merge Answer
    ↓
Re-run Verification Gate
```

核心规则：

* 先通过 Verification Gate 找出缺口（gap）。
* 再根据 gap 设计问题。
* 每次只问一个问题。
* 用户回答后，必须重新判断，不继续旧问题。

---

# ClarificationQuestionSet

用于根据当前 gap 规划“可能要问的问题”。

它不直接展示给用户。

## 简单规则

* 只针对已有 gap 设计问题。
* 一个问题可以解决多个 gap。
* 不需要固定问满 5 个问题。
* 问题只是候选，不是固定顺序。

例如：

```text
gap:
- semantic
- recall
- scope
```

可以设计：

```text
Q1: 确认真实语义
Q2: 确认代码位置
Q3: 确认适用范围
```

但实际只会先选一个最重要的问。

---

## 关键字段（简化）

| 字段                  | 含义                  |
| ------------------- | ------------------- |
| `item_id`           | 当前对象                |
| `item_kind`         | knowledge / ability |
| `question_count`    | 已问次数（最多 5）          |
| `verification_gaps` | 当前缺口                |
| `questions`         | 候选问题                |

---

## YAML 示例

```yaml
ClarificationQuestionSet:
  item_id: knowledge.xxx
  item_kind: knowledge

  question_count: 0

  verification_gaps:
    - id: gap.semantic
      type: semantic

  questions:
    - id: q.semantic
      targets: [gap.semantic]
      question: "这个接口实际做什么？"
      answer_type: single_choice
```

---

# ClarificationQuestion

从 QuestionSet 中选出的“当前要问的一个问题”。

每次只能问一个。

---

## 选择原则（简化）

优先问：

1. 阻塞问题（blocking gap）
2. 最关键语义
3. 能解决多个 gap 的问题
4. 前置问题

不要按固定顺序问。

---

## 关键字段（简化）

| 字段                | 含义    |
| ----------------- | ----- |
| `item_id`         | 当前对象  |
| `question_id`     | 问题 ID |
| `question_number` | 第几次提问 |
| `question`        | 问题内容  |
| `answer_type`     | 回答类型  |
| `options`         | 选项    |

---

## YAML 示例

```yaml
ClarificationQuestion:
  item_id: knowledge.xxx
  question_id: q.scope
  question_number: 1

  question: "这条知识适用于哪些范围？"

  answer_type: single_choice

  options:
    - label: "仅当前仓库"
      value: current_repo
    - label: "全团队"
      value: team_wide
    - label: "其他（请补充）"
      value: other
```

---

# 用户展示

内部问题必须转成自然语言再问用户。

示例：

```text
为了避免误用，我需要确认一下：这条知识适用于哪些范围？

请选择：
1. 仅当前仓库
2. 全团队
3. 其他（请补充）
```

---

# 用户回答后

流程必须重新开始：

```text
User Answer
    ↓
Merge
    ↓
Run Verification Gate
```

规则：

* 旧问题全部作废
* 重新生成 QuestionSet
* 再选一个新问题
* 最多问 5 次

---

# 约束

必须遵守：

```text
先找 gap → 再设计问题 → 每次只问一个 → 回答后重跑
```

禁止：

* 没有 gap 也提问
* 一次问多个问题
* 按固定顺序问
* 用户回答后继续旧问题
* 编造选项或事实
