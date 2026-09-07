# 部署密钥对

## 应用场景

密钥对管理服务（Key Pair Service，KPS）是华为云提供的用于管理SSH密钥对的服务，可以帮助您安全、便捷地管理ECS实例的登录凭证。通过使用Terraform，您可以自动化地创建和管理KPS密钥对，并将公钥注入到ECS实例中，实现无密码登录，提高安全性和运维效率。

本最佳实践将介绍如何使用Terraform在华为云上创建一个KPS密钥对，并配置其加密方式、所属用户、描述等信息。通过本实践，您可以了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的密钥对资源。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [密钥对管理服务密钥对（huaweicloud_kps_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)

### 资源/数据源依赖关系

```
huaweicloud_kps_keypair
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建KPS密钥对

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建KPS密钥对
variable "keypair_name" {
  description = "The name of the KPS keypair"
  type        = string
}

variable "keypair_scope" {
  description = "The scope of the KPS keypair"
  type        = string
  default     = "user"
}

variable "keypair_user_id" {
  description = "The user ID to which the KPS keypair belongs"
  type        = string
  default     = ""
}

variable "keypair_encryption_type" {
  description = "The encryption mode of the KPS keypair"
  type        = string
  default     = "kms"
}

variable "kms_key_id" {
  description = "The ID of the KMS key"
  type        = string
  default     = ""
}

variable "kms_key_name" {
  description = "The name of the KMS key"
  type        = string
  default     = ""

  validation {
    condition     = var.keypair_encryption_type != "kms" || (var.kms_key_id != "" || var.kms_key_name != "")
    error_message = "At least one of kms_key_id and kms_key_name must be provided when keypair_encryption_type set to **kms**"
  }
}

variable "keypair_description" {
  description = "The description of the KPS keypair"
  type        = string
  default     = ""
}

resource "huaweicloud_kps_keypair" "test" {
  name            = var.keypair_name
  scope           = var.keypair_scope
  user_id         = var.keypair_user_id
  encryption_type = var.keypair_encryption_type
  kms_key_id      = var.kms_key_id
  kms_key_name    = var.kms_key_name
  description     = var.keypair_description
}
```

**参数说明**：

- **name**：通过引用输入变量 keypair_name 进行赋值，用于指定密钥对的名称。
- **scope**：通过引用输入变量 keypair_scope 进行赋值，用于指定密钥对的作用范围，默认为 `user`。
- **user_id**：通过引用输入变量 keypair_user_id 进行赋值，用于指定密钥对所属的用户ID。
- **encryption_type**：通过引用输入变量 keypair_encryption_type 进行赋值，用于指定密钥对的加密方式，默认为 `kms`。
- **kms_key_id**：通过引用输入变量 kms_key_id 进行赋值，用于指定KMS密钥的ID。当加密方式为 `kms` 时，需要与 kms_key_name 至少提供一个。
- **kms_key_name**：通过引用输入变量 kms_key_name 进行赋值，用于指定KMS密钥的名称。当加密方式为 `kms` 时，需要与 kms_key_id 至少提供一个。
- **description**：通过引用输入变量 keypair_description 进行赋值，用于指定密钥对的描述信息。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
keypair_name        = "tf_test_keypair"
kms_key_id          = "your_kms_key_id"
keypair_description = "This is a KPS keypair created by Terraform"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="keypair_name=my-keypair"`
2. 环境变量：`export TF_VAR_keypair_name=my-keypair`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建KPS密钥对
4. 运行 `terraform show` 查看已创建的KPS密钥对

## 参考信息

- [华为云数据加密服务产品文档](https://support.huaweicloud.com/dew/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [数据加密服务密钥对最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dew/kps-keypair)
