# 部署锁分析

## 应用场景

数据管理服务（Data Admin Service，简称DAS）提供数据库智能运维能力，支持对数据库实例进行死锁检测与历史事务分析，帮助DBA快速定位锁等待、死锁和长事务等问题。通过开启全量死锁检测与历史事务开关，并按需将历史事务导出至OBS桶，可实现锁相关问题的持续观测与离线分析。本最佳实践将介绍如何使用Terraform自动化部署DAS锁分析相关配置，包括开启全量死锁检测、开启历史事务开关，以及创建历史事务导出任务。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [全量死锁检测开关资源（huaweicloud_das_full_dead_lock_switch）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_full_dead_lock_switch)
- [历史事务开关资源（huaweicloud_das_history_transaction_switch）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_history_transaction_switch)
- [历史事务导出任务资源（huaweicloud_das_history_transaction_export_task）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_history_transaction_export_task)

### 资源/数据源依赖关系

```text
huaweicloud_das_full_dead_lock_switch

huaweicloud_das_history_transaction_switch
    └── huaweicloud_das_history_transaction_export_task
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 开启全量死锁检测

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建全量死锁检测开关资源：

```hcl
variable "lock_analysis_instance_id" {
  description = "The ID of the database instances, separated by commas"
  type        = string
}

variable "full_dead_lock_switch_on" {
  description = "Whether to enable the full dead lock switch"
  type        = bool
  default     = false
}

variable "full_dead_lock_retention_hours" {
  description = "The retention hours of the full dead lock data"
  type        = number
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全量死锁检测开关资源
resource "huaweicloud_das_full_dead_lock_switch" "test" {
  instance_id = var.lock_analysis_instance_id

  switch_on = var.full_dead_lock_switch_on

  retention_hours = var.full_dead_lock_retention_hours
}
```

**参数说明**：
- **instance_id**：数据库实例ID（多个实例以逗号分隔），通过引用输入变量 `lock_analysis_instance_id` 进行赋值
- **switch_on**：是否开启全量死锁检测开关，通过引用输入变量 `full_dead_lock_switch_on` 进行赋值
- **retention_hours**：全量死锁数据的保留时长（小时），通过引用输入变量 `full_dead_lock_retention_hours` 进行赋值

> 注意：部署前需已存在目标数据库实例。请根据业务需要合理配置死锁数据保留时长。

### 3. 开启历史事务开关

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建历史事务开关资源：

```hcl
variable "history_transaction_status" {
  description = "The switch status of the history transaction"
  type        = string
  default     = "Enabled"
}

variable "lock_analysis_datastore_type" {
  description = "The database type"
  type        = string
  default     = "MySQL"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建历史事务开关资源
resource "huaweicloud_das_history_transaction_switch" "test" {
  instance_id    = var.lock_analysis_instance_id
  status         = var.history_transaction_status
  datastore_type = var.lock_analysis_datastore_type
}
```

**参数说明**：
- **instance_id**：数据库实例ID（多个实例以逗号分隔），通过引用输入变量 `lock_analysis_instance_id` 进行赋值
- **status**：历史事务开关状态，通过引用输入变量 `history_transaction_status` 进行赋值
- **datastore_type**：数据库类型，通过引用输入变量 `lock_analysis_datastore_type` 进行赋值

### 4. 创建历史事务导出任务

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建历史事务导出任务资源：

```hcl
variable "history_transaction_bucket_name" {
  description = "The OBS bucket name for exporting history transactions"
  type        = string
}

variable "history_transaction_start_time" {
  description = "The start time of the history transactions to export, in RFC3339 format"
  type        = string
  default     = "2000-06-01T00:00:00+08:00"
}

variable "history_transaction_end_time" {
  description = "The end time of the history transactions to export, in RFC3339 format"
  type        = string
  default     = "2099-06-02T00:00:00+08:00"
}

variable "history_transaction_file_path" {
  description = "The OBS file directory for the export task"
  type        = string
  default     = null
}

variable "history_transaction_time_zone" {
  description = "The time zone for the export task"
  type        = string
  default     = "UTC+8"
}

variable "history_transaction_order_field" {
  description = "The sort field for the export task"
  type        = string
  default     = "collectTime"
}

variable "history_transaction_order_by" {
  description = "The sort order for the export task"
  type        = string
  default     = "asc"
}

variable "history_transaction_last_sec_min" {
  description = "The minimum duration for the export task"
  type        = number
  default     = 0
}

variable "history_transaction_last_sec_max" {
  description = "The maximum duration for the export task"
  type        = number
  default     = 100
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建历史事务导出任务资源
resource "huaweicloud_das_history_transaction_export_task" "test" {
  instance_id = var.lock_analysis_instance_id
  bucket_name = var.history_transaction_bucket_name
  start_time  = var.history_transaction_start_time
  end_time    = var.history_transaction_end_time
  file_path   = var.history_transaction_file_path
  time_zone   = var.history_transaction_time_zone
  order_field = var.history_transaction_order_field
  order_by    = var.history_transaction_order_by

  last_sec_min = var.history_transaction_last_sec_min
  last_sec_max = var.history_transaction_last_sec_max

  depends_on = [huaweicloud_das_history_transaction_switch.test]
}
```

**参数说明**：
- **instance_id**：数据库实例ID（多个实例以逗号分隔），通过引用输入变量 `lock_analysis_instance_id` 进行赋值
- **bucket_name**：用于导出历史事务的OBS桶名称，通过引用输入变量 `history_transaction_bucket_name` 进行赋值
- **start_time**：待导出历史事务的开始时间（RFC3339格式），通过引用输入变量 `history_transaction_start_time` 进行赋值
- **end_time**：待导出历史事务的结束时间（RFC3339格式），通过引用输入变量 `history_transaction_end_time` 进行赋值
- **file_path**：导出任务的OBS文件目录，通过引用输入变量 `history_transaction_file_path` 进行赋值
- **time_zone**：导出任务的时区，通过引用输入变量 `history_transaction_time_zone` 进行赋值
- **order_field**：导出任务的排序字段，通过引用输入变量 `history_transaction_order_field` 进行赋值
- **order_by**：导出任务的排序方式，通过引用输入变量 `history_transaction_order_by` 进行赋值
- **last_sec_min**：导出任务的最小持续时长，通过引用输入变量 `history_transaction_last_sec_min` 进行赋值
- **last_sec_max**：导出任务的最大持续时长，通过引用输入变量 `history_transaction_last_sec_max` 进行赋值

> 注意：历史事务导出任务依赖已开启的历史事务开关。请确保OBS桶已存在，并具备相应的写入权限。

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 锁分析实例配置
lock_analysis_instance_id = "your_rds_instance_id"

# 全量死锁检测配置
full_dead_lock_switch_on       = true
full_dead_lock_retention_hours = 24

# 历史事务开关配置
history_transaction_status   = "Enabled"
lock_analysis_datastore_type = "MySQL"

# 历史事务导出任务配置
history_transaction_bucket_name = "your_obs_bucket"
history_transaction_start_time  = "2026-08-01T00:00:00Z"
history_transaction_end_time    = "2026-08-10T23:59:59Z"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="lock_analysis_instance_id=your_rds_instance_id" -var="history_transaction_bucket_name=your_obs_bucket"`
2. 环境变量：`export TF_VAR_lock_analysis_instance_id=your_rds_instance_id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建锁分析相关配置
4. 运行 `terraform show` 查看已创建的锁分析相关配置

## 参考信息

- [华为云DAS产品文档](https://support.huaweicloud.com/das/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DAS锁分析最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/das/lock-analysis)
