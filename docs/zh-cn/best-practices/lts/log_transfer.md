# 部署日志转储

## 应用场景

华为云云日志服务（LTS）的日志转储功能允许用户将日志数据从LTS日志组和日志流中定期转储到OBS（对象存储服务）中，实现日志数据的长期存储和备份。通过配置日志转储任务，您可以实现日志数据的自动化归档、成本优化和合规性要求。

本最佳实践特别适用于需要长期保存日志数据、实现日志数据备份、满足合规审计要求、优化存储成本的场景，如日志归档、数据湖构建、合规性审计等。本最佳实践将介绍如何使用Terraform自动化部署LTS日志转储，包括日志组、日志流、OBS存储桶和日志转储任务的创建，实现完整的日志转储解决方案。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [日志组资源（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [日志流资源（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [OBS存储桶资源（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [日志转储资源（huaweicloud_lts_transfer）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_transfer)

### 资源/数据源依赖关系

```
huaweicloud_lts_group
    ├── huaweicloud_lts_stream
    └── huaweicloud_lts_transfer

huaweicloud_obs_bucket
    └── huaweicloud_lts_transfer
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建日志组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建日志组资源：

```hcl
variable "group_name" {
  description = "The name of the log group"
  type        = string
}

variable "group_log_expiration_days" {
  description = "The log expiration days of the log group"
  type        = number
  default     = 14
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建日志组资源
resource "huaweicloud_lts_group" "test" {
  group_name  = var.group_name
  ttl_in_days = var.group_log_expiration_days
}
```

**参数说明**：
- **group_name**：日志组的名称，通过引用输入变量group_name进行赋值
- **ttl_in_days**：日志组的日志过期天数，通过引用输入变量group_log_expiration_days进行赋值，默认值为14

### 3. 创建日志流

在TF文件中添加以下脚本以告知Terraform创建日志流资源：

```hcl
variable "stream_name" {
  description = "The name of the log stream"
  type        = string
}

variable "stream_log_expiration_days" {
  description = "The log expiration days of the log stream"
  type        = number
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建日志流资源
resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.stream_name
  ttl_in_days = var.stream_log_expiration_days
}
```

**参数说明**：
- **group_id**：日志流所属的日志组ID，引用前面创建的日志组资源的ID
- **stream_name**：日志流的名称，通过引用输入变量stream_name进行赋值
- **ttl_in_days**：日志流的日志过期天数，通过引用输入变量stream_log_expiration_days进行赋值，默认值为null（继承日志组设置）

### 4. 创建OBS存储桶

在TF文件中添加以下脚本以告知Terraform创建OBS存储桶资源：

```hcl
variable "bucket_name" {
  description = "The name of the OBS bucket"
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
- **acl**：存储桶的ACL策略，设置为"private"表示私有访问
- **force_destroy**：是否强制删除存储桶，设置为true表示允许删除非空存储桶

### 5. 创建日志转储任务

在TF文件中添加以下脚本以告知Terraform创建日志转储资源：

```hcl
variable "transfer_type" {
  description = "The type of the log transfer"
  type        = string
  default     = "OBS"
}

variable "transfer_mode" {
  description = "The mode of the log transfer"
  type        = string
  default     = "cycle"
}

variable "transfer_storage_format" {
  description = "The storage format of the log transfer"
  type        = string
  default     = "JSON"
}

variable "transfer_status" {
  description = "The status of the log transfer"
  type        = string
  default     = "ENABLE"
}

variable "bucket_dir_prefix_name" {
  description = "The prefix path of the OBS transfer task"
  type        = string
  default     = "LTS-test/%GroupName/%StreamName/%Y/%m/%d/%H/%M"
}

variable "bucket_time_zone" {
  description = "The time zone of the OBS bucket"
  type        = string
  default     = "UTC"
}

variable "bucket_time_zone_id" {
  description = "The time zone ID of the OBS bucket"
  type        = string
  default     = "Etc/GMT"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建日志转储资源
resource "huaweicloud_lts_transfer" "test" {
  log_group_id = huaweicloud_lts_group.test.id

  log_streams {
    log_stream_id = huaweicloud_lts_stream.test.id
  }

  log_transfer_info {
    log_transfer_type   = var.transfer_type
    log_transfer_mode   = var.transfer_mode
    log_storage_format  = var.transfer_storage_format
    log_transfer_status = var.transfer_status

    log_transfer_detail {
      obs_bucket_name     = huaweicloud_obs_bucket.test.bucket
      obs_period          = 3
      obs_period_unit     = "hour"
      obs_dir_prefix_name = var.bucket_dir_prefix_name
      obs_time_zone       = var.bucket_time_zone
      obs_time_zone_id    = var.bucket_time_zone_id
    }
  }

  depends_on = [
    huaweicloud_obs_bucket.test,
  ]
}
```

**参数说明**：
- **log_group_id**：日志转储所属的日志组ID，引用前面创建的日志组资源的ID
- **log_streams**：日志流配置块
  - **log_stream_id**：要转储的日志流ID，引用前面创建的日志流资源的ID
- **log_transfer_info**：日志转储信息配置块
  - **log_transfer_type**：日志转储类型，通过引用输入变量transfer_type进行赋值，默认值为"OBS"
  - **log_transfer_mode**：日志转储模式，通过引用输入变量transfer_mode进行赋值，默认值为"cycle"（周期转储）
  - **log_storage_format**：日志存储格式，通过引用输入变量transfer_storage_format进行赋值，默认值为"JSON"
  - **log_transfer_status**：日志转储状态，通过引用输入变量transfer_status进行赋值，默认值为"ENABLE"
- **log_transfer_detail**：日志转储详细配置块
  - **obs_bucket_name**：OBS存储桶名称，引用前面创建的OBS存储桶资源的名称
  - **obs_period**：转储周期，设置为3表示每3个时间单位转储一次
  - **obs_period_unit**：转储周期单位，设置为"hour"表示按小时计算
  - **obs_dir_prefix_name**：OBS目录前缀名称，通过引用输入变量bucket_dir_prefix_name进行赋值，支持变量替换
  - **obs_time_zone**：OBS时区，通过引用输入变量bucket_time_zone进行赋值，默认值为"UTC"
  - **obs_time_zone_id**：OBS时区ID，通过引用输入变量bucket_time_zone_id进行赋值，默认值为"Etc/GMT"
- **depends_on**：显式依赖关系，确保OBS存储桶在日志转储任务创建前已存在

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 日志组配置
group_name = "tf_test_log_group"

# 日志流配置
stream_name = "tf_test_log_stream"

# OBS存储桶配置
bucket_name = "tf-test-log-transfer-obs-bucket"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="group_name=my-group" -var="stream_name=my-stream"`
2. 环境变量：`export TF_VAR_group_name=my-group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建日志转储
4. 运行 `terraform show` 查看已创建的日志转储详情

## 参考信息

- [华为云云日志服务产品文档](https://support.huaweicloud.com/lts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [LTS日志转储最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/lts/log-transfer)
