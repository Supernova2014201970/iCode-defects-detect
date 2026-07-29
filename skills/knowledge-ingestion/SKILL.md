---
name: knowledge-ingestion
description: 将用户提供的检视相关信息转化为后续代码检视系统可以持续召回、消费和演进的 `knowledge` 或 `ability` 资产。

---

# Knowledge Ingestion

## Workflow

严格按照以下流程执行：

```text
User Input
    ↓
Split Candidate Items
    ↓
ValueGate
    ↓
Knowledge / Ability
    ↓
Concern Classification
    ↓
Verification Gate
    ↓
┌──────────── sufficient ────────────┐
│                                    │
│                                 Finalize
│                                    │
│                              Store / Merge
│
└─ insufficient
        ↓
ClarificationQuestionSet
        ↓
ClarificationQuestion
        ↓
Ask User One Question
        ↓
Merge User Answer
        ↓
Re-run From ValueGate
```

对应伪代码：

```text
items = split_user_input_into_candidate_items(user_input)

for item in items:

    clarification_count = 0

    while true:

        merge_latest_user_input(item)

        value = run_ValueGate(item)

        if value == no_save:
            explain_no_save_reason(item)
            finish(item)
            break

        kind = classify_knowledge_or_ability(item)

        concern = classify_concern(
            item,
            references/classification.json
        )

        if kind == knowledge:
            verification = run_KnowledgeVerificationGate(
                item,
                concern
            )
        else:
            verification = run_AbilityVerificationGate(
                item,
                concern
            )

        if verification == sufficient:
            break

        if clarification_count >= 5:
            mark_incomplete(item)
            break

        question_set = build_ClarificationQuestionSet(
            item,
            verification
        )

        question = select_ClarificationQuestion(
            question_set,
            item
        )

        ask_one_natural_language_question(question)

        clarification_count += 1

        wait_for_user_answer()

    if kind == knowledge:
        finalize_and_store_knowledge(item, concern)

    if kind == ability:
        finalize_and_store_ability(item, concern)

output_processing_result()
```

## Core Rules

### Do

* 先判断是否值得保存，再进行分类和追问。
* 将混合输入拆分为独立 candidate item。
* 每个 item 独立判断价值、类型、concern 和完整性。
* 每次获得用户新输入后，先合并到当前 item。
* 合并后重新执行 `ValueGate`、类型判断、concern 分类和 Verification Gate。
* 信息不足时，一次只向用户询问一个问题。
* 当前 item 收敛后，再处理下一个 item。
* 只追问会改变后续检视消费方式的信息。

### Do Not

不得臆造用户没有提供、且无法从上下文合理推出的：

* anchor
* 企业事实
* 安全例外
* source / sink / sanitizer 语义
* 控制点
* 风险条件
* 证据要求
* 适用范围

不要：

* 为了字段完整而追问。
* 要求用户补充与后续检视无关的信息。
* 向用户展示未整理完成的 `InspectionKnowledge`。
* 向用户展示完整 `ClarificationQuestionSet`。
* 向用户展示 `ClarificationQuestion` 内部报文。
* 一次询问多个问题。
* 机械执行预先生成的问题列表。
* 将 Agent 自己的猜测包装成选项诱导用户确认。

---

# Input Types

用户输入可能包括：

* 企业接口语义：内部接口、RPC、SDK、wrapper、endpoint 的真实行为。
* 检查经验：某种调用或代码模式应该如何判断。
* 误报反馈：已有检查为什么误报，哪些模式实际安全。
* 漏报反馈：缺失的 source、sink、字段、配置或业务语义。
* 样例材料：代码、MR 评论、缺陷单、安全事故、人工 review 结论。
* 作用域信息：仓库、服务、团队、语言、框架、目录或版本。
* 能力想法：新的缺陷模式、检查流程或分析能力。

---

# ValueGate

首先判断当前 item 是否具有保存价值。

## Save

满足以下任一条件时，通常应保存：

* 包含企业特有接口、字段、框架或业务语义。
* 会改变知识召回方式。
* 会改变风险判断。
* 会改变 finding 证据要求。
* 能解释已有误报或漏报。
* 提供新的安全例外或 sanitizer。
* 提供新的 scope。
* 提供真实缺陷、代码、MR 或 review 样例。
* 提出新的缺陷模式。
* 提出新的分析步骤或检查流程。
* 对已有能力增加触发条件、输入信号、证据、例外或样例。

## No Save

以下信息通常不保存：

* 与企业检视无关的普通常识。
* 没有企业增量的通用漏洞定义。
* 纯观点、情绪或讨论。
* 不会改变召回、分析、判断、证据、suppress 或 scope 的信息。
* 与已有资产完全重复，且没有新的语义、证据、样例或适用范围。

判断为 `no_save` 后：

```text
do not create knowledge
do not create ability
explain briefly
finish current item
```

---

# Knowledge vs Ability

## Knowledge

`knowledge` 是事实、上下文或判断依据，可以被已有检视能力直接消费。

判断标准：

> Agent 已经具备基本检查能力，只是缺少一个事实、语义、边界或判断依据。

例如：

* 某内部接口会将参数写入数据库。
* `request.body.sql` 属于外部输入。
* `SafeSqlBuilder` 已完成参数化处理。
* 某字段属于敏感数据。
* 某模板属于安全例外。
* 某规则仅适用于仓库 A。
* 形成 finding 前必须同时看到 A 和 B。

## Ability

`ability` 是新的检查目标、缺陷模式或分析流程。

判断标准：

> 仅增加一个事实仍无法完成检查，Agent 需要新增一套识别、分析或判断方法。

例如：

* 检查一种新的缺陷模式。
* 先解析配置，再关联代码行为。
* 跨文件追踪 A 到 B。
* 联合 A、B、C 三种信号判断风险。
* 新增证据收集流程。
* 新增分析步骤或检查 workflow。

## Mixed Input

输入同时包含 knowledge 和 ability 时必须拆分。

例如：

```text
XXConfig.remote=true 表示数据来自外部系统。
以后检查时需要先解析配置，再判断数据是否进入 executeCommand。
```

拆分为：

```text
knowledge:
XXConfig.remote=true 表示数据来自外部系统

ability:
解析 XXConfig，并将配置语义关联到 executeCommand 数据流分析
```

---

# Concern Classification

分类标准来自：

```text
references/classification.json
```

分类原则参考：

```text
references/Concern-Classification.md
```

`Concern-Classification.md` 同时定义 knowledge 和 ability 的命名标准。

执行规则：

1. 优先匹配已有分类。
2. 不因为用户使用新的表述方式就创建扩展分类。
3. 只有现有三级分类无法准确表达时，才使用扩展分类。
4. 用户补充重要信息后，重新判断 concern。
5. knowledge 和 ability 使用同一套 concern 分类和命名规则。

## concern_slug

非扩展分类：

```text
一级分类-二级分类-三级分类
```

扩展分类：

```text
一级分类-二级分类-三级分类-自定义分类
```

---

# Verification Gate 原则

`KnowledgeVerificationGate` 和 `AbilityVerificationGate` 只负责判断：

> **当前信息是否存在会影响后续安全消费、创建或融合的关键缺口。**

Gate 不负责设计追问问题。识别出的缺口统一交给 `ClarificationQuestionSet`。

统一遵循以下原则：

1. **只检查关键缺口**
   不要求信息绝对完整，只检查缺失后是否会显著影响后续召回、分析、判断或融合。

2. **核心语义必须明确**
   必须知道用户表达的事实、经验、规则或检查目标是什么，不能依赖 Agent 自行补全关键语义。

3. **后续作用必须可识别**
   必须能够判断该 knowledge 或 ability 如何影响后续检视，例如增强召回、风险判断、误报抑制或增加分析行为。

4. **必须存在可识别的触发条件**
   至少能够识别一种召回或启动信号，例如 API、symbol、代码模式、配置、annotation、framework pattern 或语义模式。

5. **适用范围必须足够明确**
   必须判断该知识或能力在什么范围、什么场景下适用，例如：

   * 哪个 repository、service 或 directory
   * 哪种 language、framework 或 version
   * 哪类业务或代码场景
   * 是普遍适用还是仅针对特定场景

   如果适用范围不明确可能导致错误召回、误报或能力误用，应记录 `scope gap`。

6. **判断关键边界按需检查**
   当 knowledge 或 ability 参与风险判断、finding 或 suppress 时，检查危险条件、安全条件和证据方向是否足够明确。

7. **禁止关键事实猜测**
   如果后续消费依赖 Agent 自己推断关键事实、规则或语义，应记录 `unsupported assumption gap`。

8. **按消费需求验证，而非按字段验证**
   Gate 判断的是信息是否足够支撑后续消费，而不是结构化字段是否全部填写。

---

# KnowledgeVerificationGate

Knowledge 满足以下基本条件时，可以判定：

```text
verification = sufficient
```

判断原则：

```text
core semantic is clear
AND review effect is identifiable
AND recall condition or semantic target is identifiable
AND applicable scope is sufficiently clear
AND no critical unsupported assumption exists
AND decision-critical boundaries are sufficiently clear
```

存在关键缺口时：

```text
verification = insufficient
gaps = [
    semantic gap
    review effect gap
    recall gap
    scope gap
    decision boundary gap
    evidence gap
    unsupported assumption gap
]
```

---

# AbilityVerificationGate

Ability 可以继续创建或融合能力 skill 的基本条件：

```text
check target is clear
AND applicable scope or scenario is sufficiently clear
AND at least one trigger or input signal is identifiable
AND required analysis increment is identifiable
AND no critical unsupported assumption exists
```

重点检查：

* **Check Target**：要检查什么问题或识别什么模式。
* **Scope / Scenario**：在哪些代码仓、业务、语言、框架或场景下使用该能力。
* **Trigger / Input Signal**：什么时候启动该能力。
* **Analysis Requirement**：相比已有能力，需要增加什么分析行为。
* **Decision Requirement**：形成 finding 时，依据什么类型的信息进行判断。
* **Critical Assumption**：是否依赖 Agent 编造关键规则或事实。

存在关键缺口时，输出对应 gap，由 `ClarificationQuestionSet` 决定如何追问。


# Clarification

追问流程由三个对象组成：

```text
Verification Gate
    ↓
ClarificationQuestionSet
    ↓
ClarificationQuestion
```

三者职责不得混用。

## ClarificationQuestionSet

`ClarificationQuestionSet` 是内部追问计划。

它必须：

> **严格基于当前 Verification Gate 识别出的 gap 生成。**

不能脱离 Gate 结论额外设计与当前消费无关的问题。

例如：

```text
KnowledgeVerificationGate gaps:
- recall gap
- decision boundary gap
- scope gap
```

则 `ClarificationQuestionSet` 只围绕这些缺口规划候选问题。

它可以包含多个候选问题，用于决定未来可能需要询问的内容。

但：

* 不直接展示给用户。
* 不代表所有问题最终都会询问。
* 用户每次回答后，原 QuestionSet 可以整体失效。
* 每次重新运行 Verification Gate 后，应重新生成 QuestionSet。

结构要求见：

```text
references/Clarification.md
```

## ClarificationQuestion

`ClarificationQuestion` 是：

> **基于当前 ClarificationQuestionSet，最终选择出的当前一轮最应该先询问用户的问题。**

一次只能产生并展示一个 `ClarificationQuestion`。

选择时考虑：

1. 哪个问题能够消除最高优先级 gap。
2. 哪个答案可能同时解决多个 gap。
3. 哪个问题最可能改变 knowledge / ability 的消费行为。
4. 哪个问题应该作为其他问题的前置条件。
5. 哪个问题用户最容易准确回答。

例如：

```text
gaps:
- 不知道接口真实行为
- 不知道安全例外
- 不知道 scope
```

应该优先询问接口真实行为。

因为接口行为尚未明确时：

* 安全例外可能没有意义
* scope 问题优先级较低

## Clarification Loop

严格执行：

```text
Verification Gate
    ↓
identify gaps
    ↓
build ClarificationQuestionSet
    ↓
select one ClarificationQuestion
    ↓
render natural language
    ↓
ask user
    ↓
merge answer
    ↓
re-run ValueGate
    ↓
re-classify knowledge / ability
    ↓
re-classify concern
    ↓
re-run Verification Gate
```

不要：

```text
generate 5 questions
then ask question 1
then ask question 2
then ask question 3
```

必须根据用户每次回答动态重新规划。

## Clarification Priority

生成 QuestionSet 时优先考虑：

1. 企业语义
2. 代码锚点或召回依据
3. 风险条件
4. 安全例外
5. 证据阈值
6. 作用域

该顺序是默认优先级，不是固定顺序。

以 Verification Gate 的实际 gap 和依赖关系为准。

## Clarification Budget

每个独立 item：

```text
max_user_questions = 5
```

每向用户实际询问一次，消耗一次机会。

内部生成或重新生成 QuestionSet 不消耗提问机会。

Bulk Input 中，每个独立 item 拥有自己的 5 次提问预算。

达到 5 次后不得继续追问。

### Incomplete Knowledge

仍存在关键缺口时：

```text
status = draft
```

并限制 `review_effect`。

可以用于：

* 背景补充
* 召回提示
* 低置信分析上下文

不得仅依赖该知识：

* suppress finding
* 形成高置信 finding
* 得出强企业语义结论

### Incomplete Ability

允许创建或融合 skill，但必须明确记录：

* 未确认前提
* 当前能力边界
* 尚缺失的关键条件

不得把未确认假设写成确定规则。

---

# User Question Rendering

对用户展示时，只输出一个自然语言问题。

每个问题应：

1. 简短说明为什么需要确认。
2. 明确询问一个核心问题。
3. 优先提供 2 至 4 个可读选项。
4. 保留自由输入入口。

推荐格式：

```text
为了避免误报，我需要确认一个点：哪些情况可以认为安全，不应该报 SQL 注入？

请选择一个或多个：
1. 使用 sql_id / 服务端模板，且没有 raw body.sql
2. parameterized_mode=true，且 SQL 文本没有拼接
3. 固定枚举 SQL，不来自用户输入
4. 暂无已知安全例外
5. 以上都不符合，我补充具体情况
```

不得向用户输出：

* `ClarificationQuestionSet`
* `ClarificationQuestion`
* `InspectionKnowledge` 草稿
* YAML
* JSON
* `knowledge_id`
* `question_id`
* `round`
* `answer_type`
* `value`
* gap 名称
* 其他内部状态

---

# 存储目录

Linux：`$HOME/chrys/co-review`
windows：`%APPDATA%\chrys\co-review`

## Knowledge Storage


knowledge 使用 JSON 保存。

文件：

```text
Linux：`$HOME/chrys/co-review/knowledge/<concern_slug>.json`
windows：`%APPDATA%\chrys\co-review\knowledge\<concern_slug>.json`
```

同一个 `concern_slug` 只允许一个 knowledge JSON 文件。

### knowledge_id

每条 knowledge 创建独立 ID：

```text
knowledge.<concern_slug>.<short_semantic_name>
```

`short_semantic_name` 根据知识的核心语义生成。

要求：

* 简短
* 可读
* 描述核心语义
* 能区分同 concern 下的其他 knowledge
* 不使用无意义序号作为主要名称

### Append

如果对应 JSON 已存在：

```text
read existing JSON
create new InspectionKnowledge item
append item to items
do not merge existing items
```

如果 JSON 不存在：

```text
create JSON
initialize items
append new item
```

用户提供一条新知识时，默认生成一个新的 knowledge item。

不要自动融合已有 knowledge。

`InspectionKnowledge` 结构和字段定义见：

```text
references/InspectionKnowledge.md
```

---

## Ability Storage

ability 使用 skill 保存。

目录：

```text
Linux：`$HOME/chrys/co-review/ability/<ability_skill_name>/`
windows：`%APPDATA%\chrys\co-review\ability\<ability_skill_name>\`
```

`ability_skill_name` 根据 concern 命名规则生成。

同一 concern 只能存在一份 ability skill。

## New Ability

执行：

```text
derive ability_skill_name from concern
    ↓
search
    ↓
not exists
    ↓
invoke $skill-creator
```

创建 skill 时，向 `$skill-creator` 提供当前已经确认的信息，包括：

* concern
* 适用场景
* 检查目标
* 输入信号
* 触发条件
* 分析步骤
* 证据要求
* 安全例外
* 输出要求

只提供已经确认或可以从上下文合理推出的信息。

不得为了创建完整 skill 而编造缺失规则。

## Existing Ability

如果已有对应 skill：

1. 读取现有 `SKILL.md`。
2. 理解已有能力范围。
3. 判断用户输入提供了什么增量。
4. 将增量融合到现有 skill。

增量可能包括：

* 新触发模式
* 新输入信号
* 新分析步骤
* 新证据要求
* 新例外
* 新样例
* 新适用范围

融合规则：

* 保留已有规则。
* 保留已有示例。
* 只增加真实能力增量。
* 不为了统一文风删除原能力。
* 不创建第二份同 concern skill。

如果新旧规则存在实质冲突：

```text
stop merge
clarify conflict with user
```

不得自行选择其中一条规则。

---

# Bulk Input

用户一次提供大量信息或明显包含多个知识点、能力点时，先拆分 candidate item。

## Initial Summary

输出简短摘要，例如：

```text
初步识别到：
- 约 5 个知识点
- 2 个能力点
- 1 个可能无需保存的信息
```

数量允许是初步判断。

## Processing Plan

为每个 candidate item 建立内部计划：

* 临时标题
* 预估类型
* 预估 concern

可以向用户展示简化计划。

不要展示内部对象或 JSON。

## Sequential Processing

逐个处理 item：

```text
Item 1
    ↓
ValueGate
    ↓
Classification
    ↓
Verification
    ↓
Clarification if needed
    ↓
Storage
    ↓
Item 2
```

当前 item 需要追问时：

* 暂停当前 item。
* 一次只询问一个问题。
* 用户回答后继续处理当前 item。
* 当前 item 收敛后再进入下一个 item。

每个 item 独立拥有最多 5 次用户提问机会。

---

# Final Output

## Need Clarification

只输出当前 `ClarificationQuestion` 渲染后的自然语言问题。

不要附加：

* 当前草稿
* 内部判断
* 完整问题计划
* 后续问题列表

## Final Knowledge

knowledge 收敛后：

1. 按 `references/InspectionKnowledge.md` 创建最终 item。
2. 写入或追加：

```text
Linux：`$HOME/chrys/co-review/knowledge/<concern_slug>.json`
windows：`%APPDATA%\chrys\co-review\knowledge\<concern_slug>.json`
```

3. 已有文件时只追加新 item。
4. 不自动融合已有 knowledge。
5. 告诉用户：

   * 落盘路径
   * 新增条目数量
   * concern 分类
   * 简短知识摘要

如果当前环境无法写入文件：

* 不得声称已经落盘。
* 输出最终 JSON item。
* 明确建议写入路径。

## Final Ability

ability 收敛后：

1. 检查是否存在同 concern ability skill。
2. 不存在时调用 `$skill-creator`。
3. 已存在时融合到原 `SKILL.md`。
4. 不创建第二份同 concern skill。
5. 告诉用户：

   * 新建或融合
   * skill 路径
   * concern 分类
   * 本次能力变更摘要

如果当前环境无法创建或修改 skill：

* 不得声称已经修改。
* 输出建议的能力增量内容。
* 明确建议 skill 路径。