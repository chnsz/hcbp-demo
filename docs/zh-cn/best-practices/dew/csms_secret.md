# 部署CSMS密钥

## 应用场景

数据加密服务（Data Encryption Workshop，DEW）CSMS（云凭据管理服务）密钥是DEW服务提供的密钥管理功能，用于安全地存储和管理敏感信息，如密码、API密钥、证书等。通过创建CSMS密钥，您可以将敏感信息存储在云端，实现密钥的统一管理和安全访问，避免在代码或配置文件中硬编码敏感信息，提高应用的安全性。通过Terraform自动化创建CSMS密钥，可以确保密钥配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建CSMS密钥。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [CSMS密钥资源（huaweicloud_csms_secret）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/csms_secret)

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建CSMS密钥资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CSMS密钥资源：

```hcl
variable "secret_name" {
  description = "The name of the secret"
  type        = string
}

variable "secret_text" {
  description = "The plaintext of a text secret"
  type        = string
  sensitive   = true
}

variable "secret_type" {
  description = "The type of the secret"
  type        = string
  default     = "COMMON"
}

variable "kms_key_id" {
  description = "The ID of the KMS key used to encrypt the secret"
  type        = string
  default     = ""
}

variable "secret_description" {
  description = "The description of the secret"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the secret belongs"
  type        = string
  default     = null
}

variable "secret_tags" {
  description = "The key/value pairs to associate with the secret"
  type        = map(string)
  default     = {}
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CSMS密钥资源
resource "huaweicloud_csms_secret" "test" {
  name                  = var.secret_name
  secret_text           = var.secret_text
  secret_type           = var.secret_type
  kms_key_id            = var.kms_key_id != "" ? var.kms_key_id : null
  description           = var.secret_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.secret_tags
}
```

**参数说明**：
- **name**：密钥名称，通过引用输入变量secret_name进行赋值
- **secret_text**：密钥明文，通过引用输入变量secret_text进行赋值
- **secret_type**：密钥类型，通过引用输入变量secret_type进行赋值，默认值为"COMMON"（通用密钥）
- **kms_key_id**：用于加密密钥的KMS密钥ID，通过引用输入变量kms_key_id进行赋值，可选参数，默认值为null（使用默认KMS密钥）
- **description**：密钥描述，通过引用输入变量secret_description进行赋值，可选参数，默认值为空字符串
- **enterprise_project_id**：密钥所属的企业项目ID，通过引用输入变量enterprise_project_id进行赋值，可选参数，默认值为null
- **tags**：密钥标签，通过引用输入变量secret_tags进行赋值，可选参数，默认值为空映射

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# CSMS密钥配置
secret_name           = "tf_test_secret"
secret_text           = "your_secret_text"
secret_description    = "This is a CSMS secret created by Terraform"
enterprise_project_id = "0"

# 密钥标签配置
secret_tags = {
  owner = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是`secret_text`需要替换为实际的密钥内容
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="secret_name=my_secret" -var="secret_text=my_secret_text"`
2. 环境变量：`export TF_VAR_secret_name=my_secret` 和 `export TF_VAR_secret_text=my_secret_text`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。由于secret_text包含敏感信息，建议使用环境变量或加密的变量文件进行设置。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建CSMS密钥：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CSMS密钥
4. 运行 `terraform show` 查看已创建的CSMS密钥详情

> 注意：CSMS密钥创建后，敏感信息会被加密存储在云端，可以通过API或控制台安全地访问。密钥支持使用KMS密钥进行加密，提供更高的安全性。密钥标签可以帮助您对密钥进行分类和管理。请确保妥善保管密钥信息，不要将敏感信息提交到版本控制系统。

## 参考信息

- [华为云DEW产品文档](https://support.huaweicloud.com/dew/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CSMS密钥最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dew/csms-secret)
