# 部署账号

## 应用场景

资源治理中心（Resource Governance Center，RGC）是华为云提供的资源治理服务，支持多账号管理、组织单元管理、蓝图配置等功能，帮助您统一管理和治理云上资源。通过创建RGC账号，可以在组织单元中创建新的账号，实现账号的统一管理和治理。本最佳实践将介绍如何使用Terraform自动化部署RGC账号，包括账号基本信息、身份存储信息和组织单元信息的配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [账号资源（huaweicloud_rgc_account）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rgc_account)

### 资源/数据源依赖关系

```text
huaweicloud_rgc_account
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建账号

在TF文件（如main.tf）中添加以下脚本以创建RGC账号：

```hcl
variable "account_name" {
  description = "The name of RGC account"
  type        = string
}

variable "account_email" {
  description = "The email of RGC account"
  type        = string
}

variable "account_phone" {
  description = "The phone number of RGC account"
  type        = string
  sensitive   = true
  default     = ""
}

variable "identity_store_user_name" {
  description = "The identity store user name of RGC account"
  type        = string
}

variable "identity_store_email" {
  description = "The identity store email of RGC account"
  type        = string
}

variable "parent_organizational_unit_name" {
  description = "The parent organizational unit name of RGC account"
  type        = string
}

variable "parent_organizational_unit_id" {
  description = "The parent organizational unit ID of RGC account"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建账号资源
resource "huaweicloud_rgc_account" "test" {
  name                            = var.account_name
  email                           = var.account_email
  phone                           = var.account_phone
  identity_store_user_name        = var.identity_store_user_name
  identity_store_email            = var.identity_store_email
  parent_organizational_unit_name = var.parent_organizational_unit_name
  parent_organizational_unit_id   = var.parent_organizational_unit_id
}
```

**参数说明**：
- **name**：账号名称，通过引用输入变量 `account_name` 进行赋值
- **email**：账号邮箱，通过引用输入变量 `account_email` 进行赋值
- **phone**：账号手机号，通过引用输入变量 `account_phone` 进行赋值，可选参数
- **identity_store_user_name**：身份存储用户名，通过引用输入变量 `identity_store_user_name` 进行赋值
- **identity_store_email**：身份存储邮箱，通过引用输入变量 `identity_store_email` 进行赋值
- **parent_organizational_unit_name**：父组织单元名称，通过引用输入变量 `parent_organizational_unit_name` 进行赋值
- **parent_organizational_unit_id**：父组织单元ID，通过引用输入变量 `parent_organizational_unit_id` 进行赋值

> 注意：账号必须属于某个组织单元，需要指定父组织单元的名称和ID。身份存储信息用于账号的身份认证和访问控制。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 账号基本信息（必填）
account_name  = "tf-test-account"
account_email = "tf-test-account@terraform.com"
account_phone = "13123456789"

# 身份存储信息（必填）
identity_store_user_name = "tf-test-account"
identity_store_email     = "tf-test-account@terraform.com"

# 组织单元信息（必填）
parent_organizational_unit_name = "your-org-unit-name"
parent_organizational_unit_id   = "your-org-unit-id"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="account_name=tf-test-account" -var="account_email=tf-test-account@terraform.com"`
2. 环境变量：`export TF_VAR_account_name=tf-test-account`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建账号
4. 运行 `terraform show` 查看已创建的账号

## 参考信息

- [华为云RGC产品文档](https://support.huaweicloud.com/rgc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RGC账号最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rgc/account)
