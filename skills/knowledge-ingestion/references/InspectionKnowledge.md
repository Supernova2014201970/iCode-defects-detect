`InspectionKnowledge` 用于沉淀和表达**可被代码检视过程消费的企业知识、业务语义和检查经验**。

每组 `InspectionKnowledge` 必须归属于一个明确的 `Concern`。`Concern` 定义“关注什么问题”，`InspectionKnowledge` 补充“在企业真实代码和业务语境下，应如何发现相关检查点，以及如何判断问题”。

## 字段说明

| 字段                                            | 含义             | 使用方式                                                                                              |
| --------------------------------------------- | -------------- | ------------------------------------------------------------------------------------------------- |
| `id`                                          | 企业知识 ID        | 格式：`knowledge.<concern_slug>.<short_semantic_name>`                                               |
| `type`                                        | 知识类型           | 例如 `enterprise_api_semantic`、`sanitizer_semantic`、`false_positive_rule`、`sensitive_data_semantic` |
| `summary`                                     | 知识摘要           | 一句话概括知识的核心语义，便于人读和检索                                                                              |
| `source.user_input_summary`                   | 用户输入摘要         | 保留知识来源及原始语境，避免知识脱离用户意图                                                                            |
| `applicable_scenarios.trigger_when`           | 触发场景           | 描述何时应召回该知识                                                                                        |
| `applicable_scenarios.risk_when`              | 风险场景           | 描述在哪些条件下相关行为可能构成问题                                                                                |
| `applicable_scenarios.safe_when`              | 安全场景           | 描述哪些条件下相关行为通常可以视为安全                                                                               |
| `anchors`                                     | 召回锚点           | 用于知识检索、代码匹配和语义召回                                                                                  |
| `anchors.code_signals`                        | 代码信号           | 可包括 `symbol`、`literal`、`field`、`argument`、`endpoint`、`config` 等                                   |
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
        "applicable_scenarios": {
          "trigger_when": [],
          "risk_when": [],
          "safe_when": []
        },
        "anchors": {
          "code_signals": []
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
        "review_effect": {}
      }
    ]
  }
}
```

## 约束

1. 每个 `InspectionKnowledgeSet` 必须且只能归属于一个 `Concern`。
2. `id` 中的 `<concern_slug>` 必须与 `InspectionKnowledgeSet.concern.slug` 一致。
3. 不得根据常识臆造用户未提供且无法从上下文明确推导的企业语义、anchor、例外条件、sanitizer 能力或证据要求。
4. `precision.level` 用于表达知识的确定程度，`review_effect` 必须据此限制知识对 Review 的实际影响。
5. `InspectionKnowledge` 是 `Concern` 下的补充知识，不负责重新定义问题类型或创建新的 Concern。


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