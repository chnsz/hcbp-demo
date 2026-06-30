# 部署数据识别规则组

## 应用场景

数据治理中心（DataArts Studio）提供数据安全模块，支持通过数据识别规则对数据进行分类和敏感级别识别。数据识别规则组用于将多条数据识别规则组织在一起，便于统一管理和应用，是构建数据安全治理体系的重要组成部分。

本最佳实践适用于在DataArts Studio工作空间中创建数据密级、数据识别规则及数据识别规则组的场景，涵盖工作空间查询、数据分类查询、数据密级创建、数据识别规则创建、规则组创建及创建结果验证。本最佳实践将介绍如何使用Terraform自动化部署上述资源，实现DataArts数据安全识别规则组的Infrastructure as Code管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [DataArts Studio工作空间列表查询数据源（data.huaweicloud_dataarts_studio_workspaces）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_studio_workspaces)
- [DataArts数据分类列表查询数据源（data.huaweicloud_dataarts_security_data_categories）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_security_data_categories)
- [DataArts数据识别规则组列表查询数据源（data.huaweicloud_dataarts_security_data_recognition_rule_groups）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_security_data_recognition_rule_groups)

### 资源

- [DataArts数据密级资源（huaweicloud_dataarts_security_data_secrecy_level）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_security_data_secrecy_level)
- [DataArts数据识别规则资源（huaweicloud_dataarts_security_data_recognition_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_security_data_recognition_rule)
- [DataArts数据识别规则组资源（huaweicloud_dataarts_security_data_recognition_rule_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_security_data_recognition_rule_group)

### 资源/数据源依赖关系

```
data.huaweicloud_dataarts_studio_workspaces
    ├── data.huaweicloud_dataarts_security_data_categories
    ├── huaweicloud_dataarts_security_data_secrecy_level
    ├── huaweicloud_dataarts_security_data_recognition_rule
    ├── huaweicloud_dataarts_security_data_recognition_rule_group
    └── data.huaweicloud_dataarts_security_data_recognition_rule_groups

data.huaweicloud_dataarts_security_data_categories
    └── huaweicloud_dataarts_security_data_recognition_rule

huaweicloud_dataarts_security_data_secrecy_level
    └── huaweicloud_dataarts_security_data_recognition_rule

huaweicloud_dataarts_security_data_recognition_rule
    └── huaweicloud_dataarts_security_data_recognition_rule_group

huaweicloud_dataarts_security_data_recognition_rule_group
    └── data.huaweicloud_dataarts_security_data_recognition_rule_groups
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询DataArts Studio工作空间

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询DataArts Studio工作空间信息并解析工作空间ID。当已通过workspace_id指定工作空间ID时，可跳过工作空间查询数据源：

```hcl
variable "workspace_id" {
  description = "The ID of the workspace"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_id" {
  description = "The ID of the DataArts Studio instance to which the workspace belongs"
  type        = string
  default     = ""
  nullable    = false
}

variable "workspace_name" {
  description = "The name of the workspace used to filter results"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下DataArts Studio工作空间信息，用于创建数据识别规则组
# Query workspaces under the specified DataArts Studio instance.
data "huaweicloud_dataarts_studio_workspaces" "test" {
  count = var.workspace_id == "" ? 1 : 0

  instance_id  = var.instance_id
  name         = var.workspace_name != "" ? var.workspace_name : null
  workspace_id = var.workspace_id != "" ? var.workspace_id : null

  lifecycle {
    precondition {
      condition     = var.instance_id != ""
      error_message = "instance_id must be provided if workspace_id is omitted."
    }
  }
}

locals {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
}
```

**参数说明**：
- **count**：数据源的创建数，仅当workspace_id为空时执行工作空间查询
- **instance_id**：DataArts Studio实例ID，通过引用输入变量instance_id进行赋值
- **name**：工作空间名称，当workspace_name不为空时用于过滤查询结果
- **workspace_id**：工作空间ID，当workspace_id不为空时用于精确查询
- **lifecycle.precondition**：前置条件，当workspace_id为空时必须提供instance_id
- **local.workspace_id**：本地变量，当workspace_id不为空时使用其值，否则引用工作空间查询数据源的返回结果

### 3. 查询数据分类

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询工作空间中的数据分类信息并解析数据识别规则所需的数据分类ID：

```hcl
variable "category_ids" {
  description = "The ID list of data categories used by data recognition rules"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "data_recognition_rule_count" {
  description = "The number of data recognition rules to create and include in the rule group"
  type        = number
  default     = 1
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下DataArts Studio工作空间中的数据分类信息，用于创建数据识别规则
# Query data categories in the workspace to resolve category IDs for data recognition rules.
data "huaweicloud_dataarts_security_data_categories" "test" {
  workspace_id = local.workspace_id
}

locals {
  category_ids = length(var.category_ids) > 0 ? var.category_ids : slice([
    for category in data.huaweicloud_dataarts_security_data_categories.test.categories : category.category_id
  ], 0, var.data_recognition_rule_count)
}
```

**参数说明**：
- **workspace_id**：工作空间ID，引用本地变量local.workspace_id
- **category_ids**：本地变量，当category_ids不为空时使用其值，否则从数据分类查询结果中截取前data_recognition_rule_count个分类ID
- **category_ids**（输入变量）：数据分类ID列表，通过引用输入变量category_ids进行赋值，默认值为空列表
- **data_recognition_rule_count**：待创建的数据识别规则数量，通过引用输入变量data_recognition_rule_count进行赋值，默认值为1

### 4. 创建数据密级

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建数据密级资源，用于数据识别规则：

```hcl
variable "rule_group_name" {
  description = "The name of the data recognition rule group"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DataArts数据密级资源，用于数据识别规则
# Create data secrecy levels for data recognition rules.
resource "huaweicloud_dataarts_security_data_secrecy_level" "test" {
  count = var.data_recognition_rule_count

  workspace_id = local.workspace_id
  name         = format("%s_secrecy_level_%d", var.rule_group_name, count.index)
}
```

**参数说明**：
- **count**：资源的创建数，等于data_recognition_rule_count
- **workspace_id**：工作空间ID，引用本地变量local.workspace_id
- **name**：数据密级名称，基于rule_group_name和索引自动生成

### 5. 创建数据识别规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建数据识别规则资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DataArts数据识别规则资源
# Create data recognition rules that will be grouped together.
resource "huaweicloud_dataarts_security_data_recognition_rule" "test" {
  count = var.data_recognition_rule_count

  workspace_id     = local.workspace_id
  rule_type        = "CUSTOM"
  name             = format("%s_rule_%d", var.rule_group_name, count.index)
  secrecy_level_id = huaweicloud_dataarts_security_data_secrecy_level.test[count.index].id
  category_id      = local.category_ids[count.index]
  method           = "NONE"

  lifecycle {
    precondition {
      condition     = length(local.category_ids) >= var.data_recognition_rule_count
      error_message = "The number of available category IDs must be greater than or equal to data_recognition_rule_count."
    }
  }
}
```

**参数说明**：
- **count**：资源的创建数，等于data_recognition_rule_count
- **workspace_id**：工作空间ID，引用本地变量local.workspace_id
- **rule_type**：规则类型，设置为"CUSTOM"表示自定义规则
- **name**：规则名称，基于rule_group_name和索引自动生成
- **secrecy_level_id**：数据密级ID，引用对应索引的数据密级资源ID
- **category_id**：数据分类ID，引用本地变量local.category_ids中对应索引的值
- **method**：识别方法，设置为"NONE"
- **lifecycle.precondition**：前置条件，可用数据分类ID数量必须大于或等于data_recognition_rule_count

### 6. 创建数据识别规则组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建数据识别规则组资源：

```hcl
variable "rule_group_description" {
  description = "The description of the data recognition rule group"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DataArts数据识别规则组资源
# Create a data recognition rule group that contains the created rules.
resource "huaweicloud_dataarts_security_data_recognition_rule_group" "test" {
  depends_on = [
    huaweicloud_dataarts_security_data_recognition_rule.test,
  ]

  workspace_id = local.workspace_id
  name         = var.rule_group_name
  description  = var.rule_group_description
  rule_ids     = huaweicloud_dataarts_security_data_recognition_rule.test[*].id
}
```

**参数说明**：
- **depends_on**：显式依赖数据识别规则资源，确保规则创建完成后再创建规则组
- **workspace_id**：工作空间ID，引用本地变量local.workspace_id
- **name**：规则组名称，通过引用输入变量rule_group_name进行赋值
- **description**：规则组描述，通过引用输入变量rule_group_description进行赋值，默认值为空字符串
- **rule_ids**：规则ID列表，引用所有已创建的数据识别规则资源ID

### 7. 查询数据识别规则组

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询已创建的数据识别规则组信息，用于验证规则组创建结果：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下DataArts数据识别规则组信息，用于验证规则组创建结果
# Query the created rule group to verify the group metadata after creation.
data "huaweicloud_dataarts_security_data_recognition_rule_groups" "test" {
  depends_on = [
    huaweicloud_dataarts_security_data_recognition_rule_group.test,
  ]

  workspace_id = local.workspace_id
  name         = var.rule_group_name
}
```

**参数说明**：
- **depends_on**：显式依赖数据识别规则组资源，确保规则组创建完成后再执行查询
- **workspace_id**：工作空间ID，引用本地变量local.workspace_id
- **name**：规则组名称，通过引用输入变量rule_group_name进行赋值，用于过滤查询结果

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`terraform.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# DataArts Studio工作空间
workspace_id = "your-dataarts-studio-workspace-id"

# 数据识别规则组
rule_group_name = "tf_test_rule_group"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="rule_group_name=my-rule-group"`
2. 环境变量：`export TF_VAR_rule_group_name=my-rule-group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建数据识别规则组及相关资源
4. 运行 `terraform show` 查看已创建的数据识别规则组详情

## 参考信息

- [华为云DataArts Studio产品文档](https://support.huaweicloud.com/dataartsstudio/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DataArts数据识别规则组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dataarts/security-data-recognition-rule-group)
