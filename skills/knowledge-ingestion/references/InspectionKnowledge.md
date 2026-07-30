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

从知识中提取能够触发风险识别的关键代码标识符，保存为 `core`。

`core` 表示：

> 看到该代码特征时，系统应联想到该知识，并进一步判断是否存在问题。

因此，`core` 应选择**当前存在问题的代码对象**，而不是目标修复后的代码对象。

例如：

用户知识：

```text
CryptoUtil.decrypt 不应该继续使用，这是旧接口。
项目要求统一迁移到 CryptoUtil.decryptV2。
````

风险触发点是：

```text
CryptoUtil.decrypt
```

因此：

```json
{
  "symbol": "decrypt",
  "kind": "call"
}
```

而不是：

```json
{
  "symbol": "decryptV2",
  "kind": "call"
}
```

因为：

* `decrypt` 是需要被识别的问题点；
* `decryptV2` 是正确使用方式，不应触发该知识。

---

## Core 匹配规则

`core.symbol` 默认采用完整标识符匹配。

禁止：

* 前缀匹配；
* 子串匹配；
* 模糊匹配。

例如：

```text
decrypt

匹配：

decrypt()

不匹配：

decryptV2()
decryptLegacy()
```

---

## Core Kind

`core.kind` 为必填字段，用于限定代码节点类型。

支持枚举：

| kind         | 描述        |
| ------------ | --------- |
| `call`       | 函数或方法调用   |
| `field`      | 字段或属性访问   |
| `type`       | 类型使用      |
| `config`     | 配置项       |
| `annotation` | 注解或装饰器    |
| `literal`    | 字符串或数值字面量 |

---

## Context 提取规则

模块名、类名、对象名等辅助标识符保存到 `context`。

`context`：

* 不独立触发知识召回；
* 不作为风险判断依据；
* 仅用于辅助模型理解代码语义和判断匹配准确性。

例如：

```text
cryptoutil.decrypt()
```

其中：

```json
{
  "core": {
    "symbol": "decrypt",
    "kind": "call"
  },
  "context": {
    "symbol": "cryptoutil",
    "kind": "type"
  }
}
```

---

## 判断原则

命中 `core`：

仅表示：

> 该代码可能与该知识相关，需要进一步检查。

不表示：

> 代码已经构成问题。

最终判断必须结合：

* `core` 匹配结果；
* `context` 信息；
* 完整代码上下文；
* 知识语义。

---

## 示例

用户知识：

```text
cryptoutil.decrypt 是旧接口，不应继续使用，
项目要求统一迁移到 cryptoutil.decryptv2。
```

抽取结果：

```json
{
  "anchors": {
    "codesignals": {
      "core": [
        {
          "symbol": "decrypt",
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
```