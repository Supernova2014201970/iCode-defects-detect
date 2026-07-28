`InspectionKnowledge` 用于沉淀和表达**可被代码检视过程消费的企业知识、业务语义和检查经验**。

每组 `InspectionKnowledge` 必须归属于一个明确的 `Concern`。`Concern` 定义“关注什么问题”，`InspectionKnowledge` 补充“在企业真实代码和业务语境下，应如何发现相关检查点，以及如何判断问题”。

知识可以服务于两个阶段：

* `target_discovery`：辅助识别值得进一步检查的代码、字段、接口或配置。
* `problem_checking`：为问题判断补充企业语义、证据要求、例外条件和安全控制信息。

一条知识可以同时服务两个阶段，也可以只服务其中一个阶段。

## 字段说明

| 字段                                            | 含义             | 使用方式                                                                                              |
| --------------------------------------------- | -------------- | ------------------------------------------------------------------------------------------------- |
| `id`                                          | 企业知识 ID        | 格式：`knowledge.<concern_slug>.<short_semantic_name>`                                               |
| `type`                                        | 知识类型           | 例如 `enterprise_api_semantic`、`sanitizer_semantic`、`false_positive_rule`、`sensitive_data_semantic` |
| `summary`                                     | 知识摘要           | 一句话概括知识的核心语义，便于人读和检索                                                                              |
| `source.user_input_summary`                   | 用户输入摘要         | 保留知识来源及原始语境，避免知识脱离用户意图                                                                            |
| `used_for`                                    | 消费阶段           | 可选：`target_discovery`、`problem_checking`                                                          |
| `applicable_scenarios.trigger_when`           | 触发场景           | 描述何时应召回该知识                                                                                        |
| `applicable_scenarios.risk_when`              | 风险场景           | 描述在哪些条件下相关行为可能构成问题                                                                                |
| `applicable_scenarios.safe_when`              | 安全场景           | 描述哪些条件下相关行为通常可以视为安全                                                                               |
| `anchors`                                     | 召回锚点           | 用于知识检索、代码匹配和语义召回                                                                                  |
| `anchors.code_signals`                        | 代码信号           | 可包括 `symbol`、`literal`、`field`、`argument`、`endpoint`、`config` 等                                   |
| `target_discovery`                            | 检查点发现视图        | 仅当 `used_for` 包含 `target_discovery` 时存在                                                           |
| `target_discovery.target_hint.target_kind`    | 检查点类型          | 描述应激活的具体 target 类型                                                                                |
| `target_discovery.target_hint.priority`       | 检查优先级          | 可选：`P0`、`P1`、`P2`、`P3`                                                                            |
| `target_discovery.target_hint.trigger_reason` | 激活原因           | 说明为什么该知识或信号值得激活 target                                                                            |
| `problem_checking`                            | 问题检查视图         | 仅当 `used_for` 包含 `problem_checking` 时存在                                                           |
| `problem_checking.semantics`                  | 企业语义           | 描述 API、字段、wrapper 或业务行为在企业场景中的真实语义                                                                |
| `problem_checking.context_hints`              | 上下文补充提示        | 仅补充企业语义相关上下文，不替代 `CapabilitySpec.context_needs`                                                   |
| `problem_checking.evidence_needed`            | 证据要求           | 描述形成正式 finding 前需要确认的关键证据                                                                         |
| `problem_checking.exceptions`                 | 例外条件           | 描述哪些场景不应判定为问题                                                                                     |
| `problem_checking.suppress_policy`            | Suppress 策略    | 描述何时允许 suppress，以及哪些情况下不能 suppress                                                                |
| `problem_checking.required_controls`          | 必要控制点          | 例如 `allowlist`、`auth`、`audit`、`approval`、`rate_limit`                                             |
| `scope`                                       | 知识作用域          | 可限定仓库、服务、团队、语言、框架或目录                                                                              |
| `status`                                      | 知识状态           | `active`、`draft`、`stale`、`superseded`                                                             |
| `precision.level`                             | 知识精确度          | `exact_anchor`、`partial_anchor`、`semantic_hint`、`example_material`                                |
| `precision.reason`                            | 精确度原因          | 说明该知识为什么被赋予当前精确度等级                                                                                |
| `review_effect`                               | Review 允许产生的效果 | 必须与 `precision.level` 匹配，限制知识对检查结果的影响范围                                                           |

## 数据结构

```json
{
  "InspectionKnowledgeSet": {
    "schema_version": "1.0",
    "concern": {
      "primary_category": "",
      "secondary_category": "",
      "tertiary_category": "",
      "custom_extension": null,
      "priority_group": "",
      "is_extensible": false,
      "logical_name": "",
      "slug": ""
    },
    "items": [
      {
        "id": "knowledge.<concern_slug>.<short_semantic_name>",
        "type": "enterprise_api_semantic",
        "summary": "",
        "source": {
          "user_input_summary": ""
        },
        "used_for": [
          "target_discovery",
          "problem_checking"
        ],
        "applicable_scenarios": {
          "trigger_when": [],
          "risk_when": [],
          "safe_when": []
        },
        "anchors": {
          "code_signals": []
        },
        "target_discovery": {
          "target_hint": {
            "target_kind": "",
            "priority": "P2",
            "trigger_reason": ""
          }
        },
        "problem_checking": {
          "semantics": {},
          "context_hints": [],
          "evidence_needed": [],
          "exceptions": [],
          "suppress_policy": [],
          "required_controls": []
        },
        "scope": {
          "repos": [],
          "services": [],
          "teams": [],
          "languages": [],
          "frameworks": [],
          "directories": [],
          "note": ""
        },
        "status": "draft",
        "precision": {
          "level": "semantic_hint",
          "reason": ""
        },
        "review_effect": {
          "target_discovery": "weak_recall_only",
          "problem_checking": "auxiliary_context_only"
        }
      }
    ]
  }
}
```

## 约束

1. 每个 `InspectionKnowledgeSet` 必须且只能归属于一个 `Concern`。
2. `id` 中的 `<concern_slug>` 必须与 `InspectionKnowledgeSet.concern.slug` 一致。
3. `used_for` 决定知识允许被哪些阶段消费：

   * 不包含 `target_discovery` 时，不应生成 `target_discovery`。
   * 不包含 `problem_checking` 时，不应生成 `problem_checking`。
4. 不得根据常识臆造用户未提供且无法从上下文明确推导的企业语义、anchor、例外条件、sanitizer 能力或证据要求。
5. `precision.level` 用于表达知识的确定程度，`review_effect` 必须据此限制知识对 Review 的实际影响。
6. `InspectionKnowledge` 是 `Concern` 下的补充知识，不负责重新定义问题类型或创建新的 Concern。


# Anchor 提取规则
从知识中提取最能代表检查对象的标识符，保存为 core。

core.symbol 默认进行完整标识符匹配，禁止前缀或子串匹配。例如 decrypt 不匹配 decryptv2。

core.kind 是必填枚举字段，用于限定代码节点类型。当前枚举值：
call：函数或方法调用
field：字段或属性访问
type：类型使用
config：配置项
annotation：注解或装饰器
literal：字符串或数值字面量

模块名、类名、对象名等辅助标识符保存到 contex。它们不独立触发召回，只作为上下文交给模型判断。

命中 core 只代表召回相关知识，不代表代码已经构成问题。模型必须结合 context、完整代码和知识语义进行最终判断。

新接口、替代接口和整改方式不应保存为 anchor，应保存到 problem_checking.semantics 或整改信息中。

无法确定代码节点类型时，不得臆造 kind；应先澄清，或将知识限制为低置信语义提示。

示例
用户知识：
cryptoutil.decrypt 是旧接口，不应继续使用，项目要求统一迁移到 cryptoutil.decryptv2。

抽取结果：
{
  "anchors": {
    "codesignals": {
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
  }
}