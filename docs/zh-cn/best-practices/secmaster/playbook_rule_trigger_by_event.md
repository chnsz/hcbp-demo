# 部署剧本规则并通过事件触发

## 应用场景

安全云脑（SecMaster）是华为云原生的新一代安全运营中心，集华为云多年安全经验，基于云原生安全，提供云上资产管理、安全态势管理、安全信息和事件管理、安全编排与自动响应等能力。通过安全云脑的安全剧本功能，您可以创建自定义的安全响应流程，实现安全事件的自动识别、分析和处置。

本最佳实践将介绍如何使用Terraform自动化部署一个安全剧本规则并通过事件触发，包括工作空间查询、剧本创建、版本管理、规则配置、动作配置、审批和启用等步骤。通过事件触发机制，当满足规则条件的安全事件发生时，系统将自动执行相应的安全动作，实现安全事件的自动化响应。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [工作空间列表查询数据源（data.huaweicloud_secmaster_workspaces）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/secmaster_workspaces)
- [数据类列表查询数据源（data.huaweicloud_secmaster_data_classes）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/secmaster_data_classes)
- [工作流列表查询数据源（data.huaweicloud_secmaster_workflows）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/secmaster_workflows)

### 资源

- [安全剧本资源（huaweicloud_secmaster_playbook）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_playbook)
- [安全剧本版本资源（huaweicloud_secmaster_playbook_version）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_playbook_version)
- [安全剧本规则资源（huaweicloud_secmaster_playbook_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_playbook_rule)
- [安全剧本动作资源（huaweicloud_secmaster_playbook_action）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_playbook_action)
- [安全剧本版本动作资源（huaweicloud_secmaster_playbook_version_action）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_playbook_version_action)
- [安全剧本审批资源（huaweicloud_secmaster_playbook_approval）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_playbook_approval)
- [安全剧本启用资源（huaweicloud_secmaster_playbook_enable）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_playbook_enable)

### 资源/数据源依赖关系

```
data.huaweicloud_secmaster_workspaces
    ├── huaweicloud_secmaster_playbook
    ├── huaweicloud_secmaster_playbook_version
    ├── huaweicloud_secmaster_playbook_rule
    ├── huaweicloud_secmaster_playbook_action
    ├── huaweicloud_secmaster_playbook_version_action
    ├── huaweicloud_secmaster_playbook_approval
    ├── huaweicloud_secmaster_playbook_enable
    ├── data.huaweicloud_secmaster_data_classes
    └── data.huaweicloud_secmaster_workflows

data.huaweicloud_secmaster_data_classes
    ├── huaweicloud_secmaster_playbook_version
    └── data.huaweicloud_secmaster_workflows
        └── huaweicloud_secmaster_playbook_action

huaweicloud_secmaster_playbook
    └── huaweicloud_secmaster_playbook_version
        ├── huaweicloud_secmaster_playbook_rule
        ├── huaweicloud_secmaster_playbook_action
        └── huaweicloud_secmaster_playbook_version_action
            └── huaweicloud_secmaster_playbook_approval
                └── huaweicloud_secmaster_playbook_enable

huaweicloud_secmaster_playbook_rule
    └── huaweicloud_secmaster_playbook_action
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询工作空间信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建安全剧本资源：

```hcl
variable "workspace_id" {
  description = "SecMaster工作空间的ID"
  type        = string
  default     = ""
}

variable "workspace_name" {
  description = "SecMaster工作空间的名称"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.workspace_id != "" || var.workspace_name != ""
    error_message = "必须提供workspace_id和workspace_name中的至少一个"
  }
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合条件的工作空间信息，用于创建安全剧本资源
data "huaweicloud_secmaster_workspaces" "test" {
  count = var.workspace_id == "" ? 1 : 0

  name = var.workspace_name
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询工作空间列表，仅当 `var.workspace_id` 为空时查询工作空间列表
- **name**：工作空间的名称，通过引用输入变量 `workspace_name` 进行赋值

### 3. 创建安全剧本资源

在TF文件中添加以下脚本以告知Terraform创建安全剧本资源：

```hcl
variable "playbook_name" {
  description = "SecMaster安全剧本的名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全剧本资源
resource "huaweicloud_secmaster_playbook" "test" {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  name         = var.playbook_name

  lifecycle {
    ignore_changes = [
      workspace_id
    ]
  }
}
```

**参数说明**：
- **workspace_id**：安全剧本所属的工作空间ID，如果指定了工作空间ID则使用该值，否则根据工作空间列表查询数据源的返回结果进行赋值
- **name**：安全剧本的名称，通过引用输入变量 `playbook_name` 进行赋值
- **lifecycle**：生命周期配置块，用于控制资源的生命周期行为
  - **ignore_changes**：指定在后续应用中要忽略的属性变更，设置为忽略 `workspace_id` 的变更

### 4. 通过数据源查询数据类信息

在TF文件中添加以下脚本以告知Terraform查询数据类信息：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的数据类信息，用于创建安全剧本版本资源
data "huaweicloud_secmaster_data_classes" "test" {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
}
```

**参数说明**：
- **workspace_id**：数据类所属的工作空间ID，如果指定了工作空间ID则使用该值，否则根据工作空间列表查询数据源的返回结果进行赋值

### 5. 创建安全剧本版本资源

在TF文件中添加以下脚本以告知Terraform创建安全剧本版本资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全剧本版本资源
resource "huaweicloud_secmaster_playbook_version" "test" {
  workspace_id      = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  playbook_id       = huaweicloud_secmaster_playbook.test.id
  dataclass_id      = try(data.huaweicloud_secmaster_data_classes.test.data_classes[0].id, null)
  rule_enable       = true
  trigger_type      = "EVENT"
  dataobject_create = true
  action_strategy   = "ASYNC"

  lifecycle {
    ignore_changes = [
      workspace_id,
      dataclass_id
    ]
  }
}
```

**参数说明**：
- **workspace_id**：安全剧本版本所属的工作空间ID，如果指定了工作空间ID则使用该值，否则根据工作空间列表查询数据源的返回结果进行赋值
- **playbook_id**：安全剧本版本所属的剧本ID，引用前面创建的安全剧本资源的ID
- **dataclass_id**：安全剧本版本关联的数据类ID，根据数据类列表查询数据源的返回结果进行赋值
- **rule_enable**：是否启用规则，设置为true表示启用规则功能
- **trigger_type**：触发类型，设置为"EVENT"表示通过事件触发
- **dataobject_create**：是否创建数据对象，设置为true表示创建数据对象
- **action_strategy**：动作策略，设置为"ASYNC"表示异步执行动作
- **lifecycle**：生命周期配置块，用于控制资源的生命周期行为
  - **ignore_changes**：指定在后续应用中要忽略的属性变更，设置为忽略 `workspace_id` 和 `dataclass_id` 的变更

### 6. 创建安全剧本规则资源

在TF文件中添加以下脚本以告知Terraform创建安全剧本规则资源：

```hcl
variable "rule_expression_type" {
  description = "安全剧本规则的表达式类型"
  type        = string
  default     = "custom"
}

variable "rule_conditions" {
  description = "安全剧本的条件规则列表"
  type = list(object({
    name   = string
    detail = string
    data   = list(string)
  }))

  validation {
    condition     = length(var.rule_conditions) >= 2
    error_message = "rule_conditions的长度必须大于或等于2"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全剧本规则资源
resource "huaweicloud_secmaster_playbook_rule" "test" {
  workspace_id    = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  version_id      = huaweicloud_secmaster_playbook_version.test.id
  expression_type = var.rule_expression_type

  dynamic "conditions" {
    for_each = var.rule_conditions
    content {
      name   = conditions.value.name
      detail = conditions.value.detail
      data   = conditions.value.data
    }
  }

  logics = split(",", join(",AND,", [
    for condition in var.rule_conditions : condition.name
  ])) # 使用AND逻辑组合条件

  lifecycle {
    ignore_changes = [
      workspace_id
    ]
  }
}
```

**参数说明**：
- **workspace_id**：安全剧本规则所属的工作空间ID，如果指定了工作空间ID则使用该值，否则根据工作空间列表查询数据源的返回结果进行赋值
- **version_id**：安全剧本规则所属的版本ID，引用前面创建的安全剧本版本资源的ID
- **expression_type**：规则的表达式类型，通过引用输入变量 `rule_expression_type` 进行赋值，默认为"custom"表示自定义表达式
- **conditions**：条件配置块（动态块），根据输入变量 `rule_conditions` 动态创建
  - **name**：条件的名称，通过引用输入变量中的条件配置进行赋值
  - **detail**：条件的详细信息，通过引用输入变量中的条件配置进行赋值
  - **data**：条件的数据，通过引用输入变量中的条件配置进行赋值
- **logics**：条件的逻辑组合，使用AND逻辑将所有条件组合在一起
- **lifecycle**：生命周期配置块，用于控制资源的生命周期行为
  - **ignore_changes**：指定在后续应用中要忽略的属性变更，设置为忽略 `workspace_id` 的变更

> 注意：`rule_conditions` 的长度必须大于或等于2，且条件之间使用AND逻辑进行组合。

### 7. 通过数据源查询工作流信息

在TF文件中添加以下脚本以告知Terraform查询工作流信息：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合条件的工作流信息，用于创建安全剧本动作资源
data "huaweicloud_secmaster_workflows" "test" {
  workspace_id  = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  data_class_id = try(data.huaweicloud_secmaster_data_classes.test.data_classes[0].id, null)
}
```

**参数说明**：
- **workspace_id**：工作流所属的工作空间ID，如果指定了工作空间ID则使用该值，否则根据工作空间列表查询数据源的返回结果进行赋值
- **data_class_id**：工作流关联的数据类ID，根据数据类列表查询数据源的返回结果进行赋值

### 8. 创建安全剧本动作资源

在TF文件中添加以下脚本以告知Terraform创建安全剧本动作资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全剧本动作资源
resource "huaweicloud_secmaster_playbook_action" "test" {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  version_id   = huaweicloud_secmaster_playbook_version.test.id
  action_id    = try(data.huaweicloud_secmaster_workflows.test.workflows[0].id, null)
  name         = try(data.huaweicloud_secmaster_workflows.test.workflows[0].name, null)

  lifecycle {
    ignore_changes = [
      workspace_id,
      action_id,
      name
    ]
  }

  depends_on = [
    huaweicloud_secmaster_playbook_rule.test
  ]
}
```

**参数说明**：
- **workspace_id**：安全剧本动作所属的工作空间ID，如果指定了工作空间ID则使用该值，否则根据工作空间列表查询数据源的返回结果进行赋值
- **version_id**：安全剧本动作所属的版本ID，引用前面创建的安全剧本版本资源的ID
- **action_id**：动作的ID，根据工作流列表查询数据源的返回结果进行赋值
- **name**：动作的名称，根据工作流列表查询数据源的返回结果进行赋值
- **lifecycle**：生命周期配置块，用于控制资源的生命周期行为
  - **ignore_changes**：指定在后续应用中要忽略的属性变更，设置为忽略 `workspace_id`、`action_id` 和 `name` 的变更
- **depends_on**：显式依赖关系，指定安全剧本动作资源依赖于安全剧本规则资源，确保规则先于动作创建

### 9. 创建安全剧本版本动作资源

在TF文件中添加以下脚本以告知Terraform创建安全剧本版本动作资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全剧本版本动作资源
resource "huaweicloud_secmaster_playbook_version_action" "test" {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  version_id   = huaweicloud_secmaster_playbook_version.test.id
  status       = "APPROVING"

  depends_on = [huaweicloud_secmaster_playbook_action.test]

  lifecycle {
    ignore_changes = [
      status,
      enabled
    ]
  }
}
```

**参数说明**：
- **workspace_id**：安全剧本版本动作所属的工作空间ID，如果指定了工作空间ID则使用该值，否则根据工作空间列表查询数据源的返回结果进行赋值
- **version_id**：安全剧本版本动作所属的版本ID，引用前面创建的安全剧本版本资源的ID
- **status**：版本动作的状态，设置为"APPROVING"表示待审批状态
- **depends_on**：显式依赖关系，指定安全剧本版本动作资源依赖于安全剧本动作资源，确保动作先于版本动作创建
- **lifecycle**：生命周期配置块，用于控制资源的生命周期行为
  - **ignore_changes**：指定在后续应用中要忽略的属性变更，设置为忽略 `status` 和 `enabled` 的变更

### 10. 创建安全剧本审批资源

在TF文件中添加以下脚本以告知Terraform创建安全剧本审批资源：

```hcl
variable "approval_content" {
  description = "安全剧本版本的审批内容"
  type        = string
  default     = "Approved for production use"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全剧本审批资源
resource "huaweicloud_secmaster_playbook_approval" "test" {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  version_id   = huaweicloud_secmaster_playbook_version.test.id
  result       = "PASS"
  content      = var.approval_content

  lifecycle {
    ignore_changes = [
      workspace_id
    ]
  }

  depends_on = [
    huaweicloud_secmaster_playbook_version_action.test
  ]
}
```

**参数说明**：
- **workspace_id**：安全剧本审批所属的工作空间ID，如果指定了工作空间ID则使用该值，否则根据工作空间列表查询数据源的返回结果进行赋值
- **version_id**：安全剧本审批所属的版本ID，引用前面创建的安全剧本版本资源的ID
- **result**：审批结果，设置为"PASS"表示审批通过
- **content**：审批内容，通过引用输入变量 `approval_content` 进行赋值，默认为"Approved for production use"
- **lifecycle**：生命周期配置块，用于控制资源的生命周期行为
  - **ignore_changes**：指定在后续应用中要忽略的属性变更，设置为忽略 `workspace_id` 的变更
- **depends_on**：显式依赖关系，指定安全剧本审批资源依赖于安全剧本版本动作资源，确保版本动作先于审批创建

### 11. 创建安全剧本启用资源

在TF文件中添加以下脚本以告知Terraform创建安全剧本启用资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全剧本启用资源
resource "huaweicloud_secmaster_playbook_enable" "test" {
  workspace_id      = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  playbook_id       = huaweicloud_secmaster_playbook.test.id
  playbook_name     = huaweicloud_secmaster_playbook.test.name
  active_version_id = huaweicloud_secmaster_playbook_approval.test.id

  lifecycle {
    ignore_changes = [
      workspace_id
    ]
  }
}
```

**参数说明**：
- **workspace_id**：安全剧本启用所属的工作空间ID，如果指定了工作空间ID则使用该值，否则根据工作空间列表查询数据源的返回结果进行赋值
- **playbook_id**：要启用的安全剧本ID，引用前面创建的安全剧本资源的ID
- **playbook_name**：要启用的安全剧本名称，引用前面创建的安全剧本资源的名称
- **active_version_id**：要激活的版本ID，引用前面创建的安全剧本审批资源的ID
- **lifecycle**：生命周期配置块，用于控制资源的生命周期行为
  - **ignore_changes**：指定在后续应用中要忽略的属性变更，设置为忽略 `workspace_id` 的变更

### 12. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 工作空间配置
workspace_name   = "tf_test_workspace"
playbook_name    = "tf_test_playbook"

# 规则配置
rule_conditions  = [
  {
    name = "condition1",
    detail = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    data = [
      "environment.domain_id",
      "==",
      "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    ]
  },
  {
    name = "condition2",
    detail = "cn-xxx-x",
    data = [
      "environment.region_id",
      "==",
      "cn-xxx-x",
    ]
  }
]

# 审批配置
approval_content = "Approved for production use"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="workspace_name=my-workspace" -var="playbook_name=my-playbook"`
2. 环境变量：`export TF_VAR_workspace_name=my-workspace`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 13. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建安全剧本规则并通过事件触发
4. 运行 `terraform show` 查看已创建的安全剧本规则详情

## 参考信息

- [华为云安全编排产品文档](https://support.huaweicloud.com/secmaster/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SecMaster剧本规则并通过事件触发最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/playbook/custom-rule-and-trigger-by-event)
