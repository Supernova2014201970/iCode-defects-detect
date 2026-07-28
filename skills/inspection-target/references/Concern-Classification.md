## 分类优先级与命名规则

### 1. 分类优先级

对 `concern`、`ability` 或 `knowledge` 进行安全分类时，若同一对象可以同时归入多个类别，按以下优先级选择唯一分类：

```text
安全编码 > 安全设计 > 安全配置
```

具体对应关系如下：

* **安全编码**：对应 `编码类 / Coding`。
* **安全设计**：对应 `设计类 / Design`。
* **安全配置**：对应 `设计类 / Design` 下的 `安全配置 / Security Configuration (CWE-16)`。

即：

> 若可以归入安全编码，则优先选择安全编码；否则选择安全设计；仅当更适合作为配置类问题时，才归入安全配置。

### 2. Concern、Ability 与 Knowledge 的统一分类与命名关系

`concern`、`ability` 和 `knowledge` 使用同一套分类路径与逻辑命名规则，并统一使用英文语义的逻辑名称并使用小写（如：`coding-improper-Input-Validation-SQL-Injection`）。

对于同一个安全关注点：

* 新建 `concern` 时，其名称必须使用英文逻辑名称（如：`coding-improper-input-validation-sql-injection`）；
* 新建 `ability` 时，其 `logical_name` 必须与 `concern name` 完全一致；
* 新建 `knowledge` 时：

  * 其 **JSON 文件名必须等于 ****`concern name`**；
  * 其展示名称（或标题）为：`concern name + 基于用户知识总结的补充名称`。

因此，在逻辑层面：

```text
concern name == ability logical name == coding-improper-input-validation-sql-injection
knowledge json filename == concern name
knowledge display name = concern name + 用户知识总结名称
```

三者表示的是同一个安全关注点，并共享统一的英文逻辑命名体系：

* `concern` 与 `ability` 完全一致；
* `knowledge` 在文件层面与 `concern` 对齐，在展示层面允许扩展语义。

### 3. 逻辑命名规则

所有分类名称统一使用**英文名称对应的语义进行 slug 转换**，逻辑名称保留完整分类路径。

非扩展分类：

```text
一级分类-二级分类-三级分类
```

扩展分类：

```text
一级分类-二级分类-三级分类-自定义分类
```

例如：

```text
编码类-不正确输入校验-SQL注入
```

对应英文逻辑路径：

```text
coding-improper-input-validation-sql-injection
```

### 4. 自定义扩展分类规则

只有当选中的三级分类明确出现在对应二级分类的：

```text
可扩展三级分类 / Extensible Tertiary Categories
```

中时，才允许创建 `custom_extension`。

否则：

```text
custom_extension = null
```

禁止自行增加第四级分类。

### 5. Skill 命名规则

由于 Skill 对目录名和 frontmatter `name` 存在命名限制，`ability skill` 的：

* 目录名；
* frontmatter `name`；

统一使用 `lowercase-hyphen-slug`。

例如：

```text
coding-improper-input-validation-sql-injection
```

但在 Skill 正文中必须保留：

1. 完整中文分类路径；
2. 完整英文分类路径；
3. 逻辑名称 `logical_name`。

不得因为 Skill 的 slug 限制丢失原始分类语义。
