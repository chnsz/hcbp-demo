# 部署系统追踪器

## 应用场景

云审计服务（Cloud Trace Service，CTS）的系统追踪器（System Tracker）是一种用于记录和存储云资源操作审计日志的核心服务。通过系统追踪器，您可以实现安全审计、合规监控、操作记录、问题排查等功能。

系统追踪器特别适用于需要记录云资源操作历史、进行安全审计、满足合规要求等场景，如企业级安全监控、合规性检查、操作审计、问题追踪等。本最佳实践将介绍如何使用Terraform自动化部署一个CTS系统追踪器，实现云资源操作的自动记录和存储。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

本最佳实践未使用数据源。

### 资源

- [OBS桶资源（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [CTS系统追踪器资源（huaweicloud_cts_tracker）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cts_tracker)

### 资源/数据源依赖关系

```
huaweicloud_obs_bucket
    └── huaweicloud_cts_tracker
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../docs/introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建OBS存储桶

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建OBS存储桶资源：

```hcl
# Variable definitions for OBS resources
variable "bucket_name" {
  description = "The name of the OBS bucket for storing trace files"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建OBS存储桶资源
resource "huaweicloud_obs_bucket" "test" {
  bucket        = var.bucket_name
  acl           = "private"
  force_destroy = true
}
```

**参数说明**：
- **bucket**：OBS存储桶的名称，通过引用输入变量bucket_name进行赋值
- **acl**：存储桶的访问控制列表，设置为"private"表示私有访问
- **force_destroy**：是否强制删除存储桶，设置为true表示允许删除非空存储桶

### 3. 创建CTS系统追踪器

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CTS系统追踪器资源：

```hcl
# Variable definitions for CTS resources
variable "tracker_enabled" {
  description = "Whether to enable the system tracker"
  type        = bool
  default     = true
}

variable "tracker_tags" {
  description = "The tags of the system tracker"
  type        = map(string)
  default     = {}
}

variable "is_system_tracker_delete" {
  description = "Whether to delete the system tracker when the tracker resource is deleted"
  type        = bool
  default     = true
}

variable "trace_object_prefix" {
  description = "The prefix of the trace object in the OBS bucket"
  type        = string
}

variable "trace_file_compression_type" {
  description = "The compression type of the trace file"
  type        = string
  default     = "gzip"
}

variable "is_lts_enabled" {
  description = "Whether to enable the trace analysis for LTS service"
  type        = bool
  default     = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CTS系统追踪器资源
resource "huaweicloud_cts_tracker" "test" {
  enabled        = var.tracker_enabled
  tags           = var.tracker_tags
  delete_tracker = var.is_system_tracker_delete
  bucket_name    = huaweicloud_obs_bucket.test.bucket
  file_prefix    = var.trace_object_prefix
  compress_type  = var.trace_file_compression_type
  lts_enabled    = var.is_lts_enabled
}
```

**参数说明**：
- **enabled**：是否启用系统追踪器，通过引用输入变量tracker_enabled进行赋值，默认值为true表示启用
- **tags**：系统追踪器的标签，通过引用输入变量tracker_tags进行赋值，用于资源分类和管理
- **delete_tracker**：删除追踪器资源时是否删除系统追踪器，通过引用输入变量is_system_tracker_delete进行赋值，默认值为true表示删除
- **bucket_name**：存储跟踪数据的OBS桶名称，通过引用OBS桶的bucket属性进行赋值
- **file_prefix**：跟踪对象在OBS桶中的前缀，通过引用输入变量trace_object_prefix进行赋值
- **compress_type**：跟踪文件的压缩类型，通过引用输入变量trace_file_compression_type进行赋值，默认值为"gzip"表示使用gzip压缩
- **lts_enabled**：是否启用LTS服务的跟踪分析，通过引用输入变量is_lts_enabled进行赋值，默认值为true表示启用

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# OBS存储桶配置
bucket_name = "tf-test-trace-bucket"

# CTS系统追踪器配置
trace_object_prefix = "tf_test"
tracker_tags        = {
  "owner" = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="bucket_name=my-bucket" -var="trace_object_prefix=my-prefix"`
2. 环境变量：`export TF_VAR_bucket_name=my-bucket`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建OBS存储桶和CTS系统追踪器
4. 运行 `terraform show` 查看已创建的OBS存储桶和CTS系统追踪器

## 参考信息

- [华为云CTS产品文档](https://support.huaweicloud.com/cts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CTS系统追踪器最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cts/system-tracker)
