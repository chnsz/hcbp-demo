# 部署账号注册

## 应用场景

资源治理中心（Resource Governance Center，RGC）是华为云提供的资源治理服务，支持多账号管理、组织单元管理、蓝图配置等功能，帮助您统一管理和治理云上资源。通过账号注册功能，可以将账号注册到组织单元中，并配置蓝图以实现资源的自动化部署和管理。本最佳实践将介绍如何使用Terraform自动化部署RGC账号注册，包括创建组织单元（可选）和账号注册（带蓝图配置）。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [组织单元资源（huaweicloud_rgc_organizational_unit）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rgc_organizational_unit)
- [账号注册资源（huaweicloud_rgc_account_enroll）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rgc_account_enroll)

### 资源/数据源依赖关系

```text
huaweicloud_rgc_organizational_unit
    └── huaweicloud_rgc_account_enroll
```

> 注意：组织单元的创建是可选的。如果 `create_organizational_unit` 为 `false`，则使用现有的 `parent_organizational_unit_id`。账号注册需要指定父组织单元ID，可以引用新创建的组织单元ID，也可以使用现有的组织单元ID。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建组织单元（可选）

在TF文件（如main.tf）中添加以下脚本以创建组织单元（可选）：

```hcl
variable "organizational_unit_name" {
  description = "The name of the organizational unit to be created (required if create_organizational_unit is true)"
  type        = string
  default     = ""
}

variable "parent_organizational_unit_id" {
  description = "The ID of the parent organizational unit. Required for account enrollment and OU creation"
  type        = string
}

variable "create_organizational_unit" {
  description = "Whether to create a new organizational unit. If false, use existing parent_organizational_unit_id"
  type        = bool
  default     = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建组织单元资源
resource "huaweicloud_rgc_organizational_unit" "test" {
  count = var.create_organizational_unit ? 1 : 0

  organizational_unit_name      = var.organizational_unit_name
  parent_organizational_unit_id = var.parent_organizational_unit_id
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建组织单元，仅当 `var.create_organizational_unit` 为 `true` 时创建组织单元
- **organizational_unit_name**：组织单元的名称，通过引用输入变量 `organizational_unit_name` 进行赋值
- **parent_organizational_unit_id**：父组织单元的ID，通过引用输入变量 `parent_organizational_unit_id` 进行赋值

### 3. 创建账号注册

在TF文件（如main.tf）中添加以下脚本以创建账号注册（带蓝图配置）：

```hcl
variable "blueprint_managed_account_id" {
  description = "The ID of the account to be enrolled with blueprint configuration"
  type        = string
}

variable "blueprint_product_id" {
  description = "The ID of the blueprint product"
  type        = string
}

variable "blueprint_product_version" {
  description = "The version of the blueprint product"
  type        = string
}

variable "blueprint_variables" {
  description = "The variables for the blueprint configuration (JSON string format)"
  type        = string
}

variable "is_blueprint_has_multi_account_resource" {
  description = "Whether the blueprint has multi-account resources"
  type        = bool
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建账号注册资源
resource "huaweicloud_rgc_account_enroll" "test" {
  managed_account_id            = var.blueprint_managed_account_id
  parent_organizational_unit_id = var.create_organizational_unit ? huaweicloud_rgc_organizational_unit.test[0].organizational_unit_id : var.parent_organizational_unit_id

  blueprint {
    blueprint_product_id                    = var.blueprint_product_id
    blueprint_product_version               = var.blueprint_product_version
    variables                               = var.blueprint_variables
    is_blueprint_has_multi_account_resource = var.is_blueprint_has_multi_account_resource
  }
}
```

**参数说明**：
- **managed_account_id**：要注册的账号ID，通过引用输入变量 `blueprint_managed_account_id` 进行赋值
- **parent_organizational_unit_id**：父组织单元的ID，如果创建了组织单元则引用新创建的组织单元ID，否则使用现有的组织单元ID
- **blueprint**：蓝图配置块，包含以下参数：
  - **blueprint_product_id**：蓝图产品ID，通过引用输入变量 `blueprint_product_id` 进行赋值
  - **blueprint_product_version**：蓝图产品版本，通过引用输入变量 `blueprint_product_version` 进行赋值
  - **variables**：蓝图变量，通过引用输入变量 `blueprint_variables` 进行赋值，需要是JSON字符串格式
  - **is_blueprint_has_multi_account_resource**：蓝图是否包含多账号资源，通过引用输入变量 `is_blueprint_has_multi_account_resource` 进行赋值

> 注意：蓝图变量需要是JSON字符串格式，例如：`"{\"environment\":\"production\",\"region\":\"cn-north-4\"}"`。如果蓝图包含多账号资源，需要将 `is_blueprint_has_multi_account_resource` 设置为 `true`。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 父组织单元ID（必填）
# 账号注册和组织单元创建都需要此参数
# 请替换为您的实际父组织单元ID
parent_organizational_unit_id = "ou-xxxxxxxxxxxxx"

# 要注册的账号ID（必填）
# 请替换为您的实际账号ID
blueprint_managed_account_id = "account-xxxxxxxxxxxxx"

# 蓝图产品配置（必填）
# 请替换为您的实际蓝图产品ID和版本
blueprint_product_id      = "blueprint-xxxxxxxxxxxxx"
blueprint_product_version = "1.0.0"

# 蓝图变量（必填）
# 根据您的蓝图需求自定义这些变量
blueprint_variables = "{\"environment\":\"production\",\"region\":\"cn-north-4\"}"

# 蓝图是否包含多账号资源（必填）
is_blueprint_has_multi_account_resource = false
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="parent_organizational_unit_id=ou-xxxxxxxxxxxxx" -var="blueprint_managed_account_id=account-xxxxxxxxxxxxx"`
2. 环境变量：`export TF_VAR_parent_organizational_unit_id=ou-xxxxxxxxxxxxx`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建账号注册
4. 运行 `terraform show` 查看已创建的账号注册

## 参考信息

- [华为云RGC产品文档](https://support.huaweicloud.com/rgc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RGC账号注册最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rgc/account-enroll)
