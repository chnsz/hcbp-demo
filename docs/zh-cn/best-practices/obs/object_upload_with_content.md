# 部署通过静态文本上传的桶对象

## 应用场景

对象存储服务（Object Storage Service，OBS）是华为云提供的高可用、高可靠、高性能、安全、低成本的对象存储服务。OBS提供海量、安全、高可靠、低成本的数据存储能力，支持多种存储类型，包括标准存储、低频存储、归档存储等，满足不同业务场景的存储需求。

OBS桶对象是OBS服务的核心资源，用于存储各种类型的数据文件。通过静态文本上传桶对象，企业可以直接将文本内容作为对象存储到OBS桶中，无需创建本地文件。这种方式特别适用于配置文件、脚本文件、文档内容等文本数据的存储和管理。静态文本上传支持多种内容类型，提供灵活的存储选项，为企业提供简单、高效的数据存储解决方案。本最佳实践将介绍如何使用Terraform自动化部署通过静态文本上传的桶对象，包括KMS密钥创建、OBS桶配置和对象上传。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [KMS密钥资源（huaweicloud_kms_key）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kms_key)
- [OBS桶资源（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [OBS桶对象资源（huaweicloud_obs_bucket_object）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket_object)

### 资源/数据源依赖关系

```
huaweicloud_kms_key.test
    └── huaweicloud_obs_bucket.test
        └── huaweicloud_obs_bucket_object.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建KMS密钥

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建KMS密钥资源：

```hcl
variable "bucket_encryption" {
  description = "OBS桶是否启用加密"
  type        = bool
  default     = true
}

variable "bucket_encryption_key_id" {
  description = "OBS桶加密密钥ID"
  type        = string
  default     = ""
  nullable    = false
}

variable "key_alias" {
  description = "KMS密钥别名"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.key_alias != "" && var.bucket_encryption && var.bucket_encryption_key_id == ""
    error_message = "当bucket_encryption为true且bucket_encryption_key_id未设置时，必须设置key_alias。"
  }
}

variable "key_usage" {
  description = "KMS密钥用途"
  type        = string
  default     = "ENCRYPT_DECRYPT"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建KMS密钥资源
resource "huaweicloud_kms_key" "test" {
  count = var.bucket_encryption && var.bucket_encryption_key_id == "" ? 1 : 0

  key_alias = var.key_alias
  key_usage = var.key_usage
}
```

**参数说明**：
- **key_alias**：密钥别名，通过引用输入变量key_alias进行赋值
- **key_usage**：密钥用途，通过引用输入变量key_usage进行赋值
- **count**：条件创建，当启用加密且未指定密钥ID时创建

### 3. 创建OBS桶

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建OBS桶资源：

```hcl
variable "bucket_name" {
  description = "OBS桶名称"
  type        = string
}

variable "bucket_storage_class" {
  description = "OBS桶存储类型"
  type        = string
  default     = "STANDARD"
}

variable "bucket_acl" {
  description = "OBS桶访问控制列表"
  type        = string
  default     = "private"
}

variable "bucket_sse_algorithm" {
  description = "OBS桶服务端加密算法"
  type        = string
  default     = "kms"
}

variable "bucket_force_destroy" {
  description = "OBS桶是否强制删除"
  type        = bool
  default     = true
}

variable "bucket_tags" {
  description = "OBS桶标签"
  type        = map(string)
  default     = {}
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建OBS桶资源
resource "huaweicloud_obs_bucket" "test" {
  bucket        = var.bucket_name
  storage_class = var.bucket_storage_class
  acl           = var.bucket_acl
  encryption    = var.bucket_encryption
  sse_algorithm = var.bucket_encryption ? var.bucket_sse_algorithm : null
  kms_key_id    = var.bucket_encryption ? var.bucket_encryption_key_id != "" ? var.bucket_encryption_key_id : huaweicloud_kms_key.test[0].id : null
  force_destroy = var.bucket_force_destroy
  tags          = var.bucket_tags

  lifecycle {
    ignore_changes = [
      sse_algorithm
    ]
  }
}
```

**参数说明**：
- **bucket**：桶名称，通过引用输入变量bucket_name进行赋值
- **storage_class**：存储类型，通过引用输入变量bucket_storage_class进行赋值
- **acl**：访问控制列表，通过引用输入变量bucket_acl进行赋值
- **encryption**：是否启用加密，通过引用输入变量bucket_encryption进行赋值
- **sse_algorithm**：服务端加密算法，当启用加密时使用kms算法
- **kms_key_id**：KMS密钥ID，优先使用输入变量，如果为空则使用创建的KMS密钥
- **force_destroy**：强制删除，通过引用输入变量bucket_force_destroy进行赋值
- **tags**：标签，通过引用输入变量bucket_tags进行赋值

### 4. 创建OBS桶对象

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建OBS桶对象资源：

```hcl
variable "object_extension_name" {
  description = "OBS对象扩展名"
  type        = string
  default     = ".txt"
  nullable    = false
}

variable "object_name" {
  description = "OBS对象名称"
  type        = string
}

variable "object_upload_content" {
  description = "OBS对象上传内容"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建OBS桶对象资源
resource "huaweicloud_obs_bucket_object" "test" {
  bucket       = huaweicloud_obs_bucket.test.id
  key          = var.object_extension_name != "" ? format("%s%s", var.object_name, var.object_extension_name) : var.object_name
  content_type = "application/xml"
  content      = var.object_upload_content
}
```

**参数说明**：
- **bucket**：桶ID，通过引用OBS桶资源（huaweicloud_obs_bucket.test）的ID进行赋值
- **key**：对象键名，根据扩展名和对象名称组合生成
- **content_type**：内容类型，设置为application/xml
- **content**：对象内容，通过引用输入变量object_upload_content进行赋值

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# KMS密钥配置
key_alias   = "tf-test-obs-key"
key_usage   = "ENCRYPT_DECRYPT"

# OBS桶配置
bucket_name        = "tf-test-obs-bucket"
bucket_storage_class = "STANDARD"
bucket_acl         = "private"
bucket_encryption  = true
bucket_sse_algorithm = "kms"
bucket_force_destroy = true
bucket_tags        = {
  environment = "test"
  created_by  = "terraform"
}

# OBS对象配置
object_extension_name = ".txt"
object_name           = "tf-test-obs-object"
object_upload_content = <<EOT
def main():
    print("Hello, World!")

if __name__ == "__main__":
    main()
EOT
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="bucket_name=my-bucket" -var="object_name=my-object"`
2. 环境变量：`export TF_VAR_bucket_name=my-bucket`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建桶对象
4. 运行 `terraform show` 查看已创建的桶对象

## 参考信息

- [华为云OBS产品文档](https://support.huaweicloud.com/obs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [OBS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/obs)
