# 部署密钥对

## 应用场景

数据加密服务（Data Encryption Workshop，DEW）KPS（密钥对管理服务）密钥对是DEW服务提供的密钥对管理功能，用于创建和管理SSH密钥对，为ECS实例等云资源提供安全的登录认证方式。通过创建KPS密钥对，您可以生成SSH密钥对，将公钥注入到ECS实例中，实现无密码登录，提高安全性和便捷性。通过Terraform自动化创建KPS密钥对，可以确保密钥对配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建KPS密钥对。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [KPS密钥对资源（huaweicloud_kps_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建KPS密钥对资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建KPS密钥对资源：

```hcl
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
    error_message = "At least one of kms_key_id and kms_key_name must be provided when keypair_encryption_type set to kms"
  }
}

variable "keypair_description" {
  description = "The description of the KPS keypair"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建KPS密钥对资源
resource "huaweicloud_kps_keypair" "test" {
  name            = var.keypair_name
  scope           = var.keypair_scope
  user_id         = var.keypair_user_id != "" ? var.keypair_user_id : null
  encryption_type = var.keypair_encryption_type
  kms_key_id      = var.kms_key_id != "" ? var.kms_key_id : null
  kms_key_name    = var.kms_key_name != "" ? var.kms_key_name : null
  description     = var.keypair_description
}
```

**参数说明**：
- **name**：密钥对名称，通过引用输入变量keypair_name进行赋值
- **scope**：密钥对作用域，通过引用输入变量keypair_scope进行赋值，默认值为"user"（用户级）
- **user_id**：密钥对所属的用户ID，通过引用输入变量keypair_user_id进行赋值，可选参数，默认值为null
- **encryption_type**：密钥对加密模式，通过引用输入变量keypair_encryption_type进行赋值，默认值为"kms"（使用KMS加密），可选值：default（默认加密）、kms（KMS加密）
- **kms_key_id**：KMS密钥ID，通过引用输入变量kms_key_id进行赋值，当encryption_type为"kms"时，kms_key_id和kms_key_name至少提供一个，可选参数，默认值为null
- **kms_key_name**：KMS密钥名称，通过引用输入变量kms_key_name进行赋值，当encryption_type为"kms"时，kms_key_id和kms_key_name至少提供一个，可选参数，默认值为null
- **description**：密钥对描述，通过引用输入变量keypair_description进行赋值，可选参数，默认值为空字符串

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# KPS密钥对配置
keypair_name        = "tf_test_keypair"
kms_key_id          = "your_kms_key_id"
keypair_description = "This is a KPS keypair created by Terraform"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是`kms_key_id`需要替换为实际的KMS密钥ID（当encryption_type为"kms"时）
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="keypair_name=my_keypair" -var="kms_key_id=my_kms_key_id"`
2. 环境变量：`export TF_VAR_keypair_name=my_keypair` 和 `export TF_VAR_kms_key_id=my_kms_key_id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。当encryption_type设置为"kms"时，必须提供kms_key_id或kms_key_name中的至少一个。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建KPS密钥对：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建KPS密钥对
4. 运行 `terraform show` 查看已创建的KPS密钥对详情

> 注意：KPS密钥对创建后，可以将公钥注入到ECS实例中，实现无密码登录。密钥对支持使用KMS密钥进行加密，提供更高的安全性。密钥对创建过程大约需要5分钟。请确保妥善保管私钥信息，不要将私钥提交到版本控制系统。

## 参考信息

- [华为云DEW产品文档](https://support.huaweicloud.com/dew/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [密钥对最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dew/kps-keypair)
