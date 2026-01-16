# 部署跨账号资源分享

## 应用场景

资源访问管理（Resource Access Manager，RAM）是华为云提供的资源共享服务，支持跨账号的资源共享和管理，帮助您实现资源的统一管理和访问控制。通过跨账号资源分享功能，可以将VPC、子网、安全组等网络资源以及ECS、RDS等计算和存储资源分享给其他账号或组织，实现资源的统一管理和访问控制。本最佳实践将介绍如何使用Terraform自动化部署跨账号资源分享，包括资源分享实例的创建、主体配置、资源URN配置和权限配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [资源分享资源（huaweicloud_ram_resource_share）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ram_resource_share)

### 资源/数据源依赖关系

```text
huaweicloud_ram_resource_share
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建资源分享

在TF文件（如main.tf）中添加以下脚本以创建资源分享：

```hcl
variable "resource_share_name" {
  description = "The name of the resource share"
  type        = string
}

variable "description" {
  description = "The description of the resource share"
  type        = string
  default     = ""
}

variable "principals" {
  description = "The list of one or more principals (account IDs or organization IDs) to share resources with"
  type        = list(string)
}

variable "resource_urns" {
  description = "The list of URNs of one or more resources to be shared. If not specified, URNs will be automatically generated from created resources (VPC, subnet, security group)"
  type        = set(string)
  default     = []
}

variable "permission_ids" {
  description = "The list of RAM permissions associated with the resource share"
  type        = list(string)
  default     = []
}

variable "allow_external_principals" {
  description = "Whether resources can be shared with any accounts outside the organization"
  type        = bool
  default     = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建资源分享资源
resource "huaweicloud_ram_resource_share" "test" {
  name                      = var.resource_share_name
  description               = var.description
  principals                = var.principals
  resource_urns             = var.resource_urns
  permission_ids            = var.permission_ids
  allow_external_principals = var.allow_external_principals
}
```

**参数说明**：
- **name**：资源分享名称，通过引用输入变量 `resource_share_name` 进行赋值
- **description**：资源分享描述，通过引用输入变量 `description` 进行赋值，可选参数
- **principals**：主体列表，通过引用输入变量 `principals` 进行赋值，用于指定要分享资源的账号ID或组织ID列表
- **resource_urns**：资源URN列表，通过引用输入变量 `resource_urns` 进行赋值，用于指定要分享的资源URN列表，可选参数
- **permission_ids**：权限ID列表，通过引用输入变量 `permission_ids` 进行赋值，用于指定与资源分享关联的RAM权限列表，可选参数
- **allow_external_principals**：是否允许与组织外的任何账号共享资源，通过引用输入变量 `allow_external_principals` 进行赋值，默认为 `false`

> 注意：资源URN是资源的唯一标识符，格式为 `资源类型:区域:账号ID:资源类型:资源ID`。如果不指定资源URN，系统会根据创建的资源（VPC、子网、安全组等）自动生成。主体可以是账号ID或组织ID，用于指定要分享资源的接收方。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 资源分享基本信息（必填）
resource_share_name = "cross-account-vpc-share"
description         = "Share VPC resources with other accounts in the organization"

# 主体列表：要分享资源的账号ID或组织ID（必填）
# 应替换为真实的账号ID
principals = [
  "01234567890123456789012345678901",
  "98765432109876543210987654321098"
]

# 要分享的资源URN列表（可选）
# 应替换为真实的资源URN
resource_urns = [
  "vpc:cn-north-4:8f06724e5c6f41f59d3e2f3ad897bb4d:subnet:5de72eeb-7977-4602-8186-8766982d9bcc"
]

# 与资源分享关联的RAM权限ID列表（可选）
# 应替换为真实的权限ID
permission_ids = [
  "f5153698-ca8b-4b3c-a839-13ff71f67885"
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="resource_share_name=cross-account-vpc-share" -var="principals=['01234567890123456789012345678901']"`
2. 环境变量：`export TF_VAR_resource_share_name=cross-account-vpc-share`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建资源分享
4. 运行 `terraform show` 查看已创建的资源分享

## 参考信息

- [华为云RAM产品文档](https://support.huaweicloud.com/ram/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RAM跨账号资源分享最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ram/cross-account-resource-share)
