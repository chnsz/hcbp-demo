# 部署KMS密钥

## 应用场景

数据加密服务（Data Encryption Workshop，DEW）KMS（密钥管理服务）密钥是DEW服务提供的密钥管理功能，用于创建和管理加密密钥，为云上数据和应用提供加密保护。通过创建KMS密钥，您可以生成和管理用于数据加密的密钥，支持多种加密算法和密钥类型，实现数据的加密存储和传输，保障数据安全。通过Terraform自动化创建KMS密钥，可以确保密钥配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建KMS密钥。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [KMS密钥资源（huaweicloud_kms_key）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kms_key)

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建KMS密钥资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建KMS密钥资源：

```hcl
variable "key_name" {
  description = "The alias name of the KMS key"
  type        = string
}

variable "key_algorithm" {
  description = "The generation algorithm of the KMS key"
  type        = string
  default     = "AES_256"
}

variable "key_usage" {
  description = "The usage of the KMS key"
  type        = string
  default     = "ENCRYPT_DECRYPT"
}

variable "key_source" {
  description = "The source of the KMS key"
  type        = string
  default     = "kms"
}

variable "key_description" {
  description = "The description of the KMS key"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the KMS key belongs"
  type        = string
  default     = null
}

variable "key_tags" {
  description = "The key/value pairs to associate with the KMS key"
  type        = map(string)
  default     = {}
}

variable "key_schedule_time" {
  description = "The number of days after which the KMS key is scheduled to be deleted"
  type        = string
  default     = "7"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建KMS密钥资源
resource "huaweicloud_kms_key" "test" {
  key_alias             = var.key_name
  key_algorithm         = var.key_algorithm
  key_usage             = var.key_usage
  origin                = var.key_source
  key_description       = var.key_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.key_tags
  pending_days          = var.key_schedule_time
}
```

**参数说明**：
- **key_alias**：密钥别名，通过引用输入变量key_name进行赋值
- **key_algorithm**：密钥生成算法，通过引用输入变量key_algorithm进行赋值，默认值为"AES_256"（AES-256算法）
- **key_usage**：密钥用途，通过引用输入变量key_usage进行赋值，默认值为"ENCRYPT_DECRYPT"（加密解密）
- **origin**：密钥来源，通过引用输入变量key_source进行赋值，默认值为"kms"（KMS生成）
- **key_description**：密钥描述，通过引用输入变量key_description进行赋值，可选参数，默认值为空字符串
- **enterprise_project_id**：密钥所属的企业项目ID，通过引用输入变量enterprise_project_id进行赋值，可选参数，默认值为null
- **tags**：密钥标签，通过引用输入变量key_tags进行赋值，可选参数，默认值为空映射
- **pending_days**：密钥计划删除天数，通过引用输入变量key_schedule_time进行赋值，默认值为"7"（7天后删除）

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# KMS密钥配置
key_name        = "tf_test_kms_key"
key_description = "This is a KMS key created by Terraform"

# 密钥标签配置
key_tags = {
  owner = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="key_name=my_kms_key" -var="key_algorithm=AES_256"`
2. 环境变量：`export TF_VAR_key_name=my_kms_key` 和 `export TF_VAR_key_algorithm=AES_256`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建KMS密钥：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建KMS密钥
4. 运行 `terraform show` 查看已创建的KMS密钥详情

> 注意：KMS密钥创建后，可以用于加密和解密数据。密钥支持多种加密算法，如AES_256、SM4等。密钥支持计划删除功能，可以设置密钥在指定天数后自动删除。密钥标签可以帮助您对密钥进行分类和管理。请确保妥善保管密钥信息，不要将敏感信息提交到版本控制系统。

## 参考信息

- [华为云DEW产品文档](https://support.huaweicloud.com/dew/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [KMS密钥最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dew/kms-key)
