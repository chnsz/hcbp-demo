# 部署日志分析

## 应用场景

数据管理服务（Data Admin Service，简称DAS）提供数据库日志分析能力，支持慢日志导出与Binlog解析，帮助开发人员和DBA定位慢SQL、回溯数据变更并开展问题排查。通过将慢日志与Binlog解析结果导出至OBS桶，可实现日志数据的离线分析与长期留存。本最佳实践将介绍如何使用Terraform自动化部署DAS日志分析相关配置，包括查询数据库用户、创建慢日志导出任务、创建Binlog解析任务，以及导出Binlog解析结果。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [数据库用户列表查询数据源（data.huaweicloud_das_database_users）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/das_database_users)

### 资源

- [慢日志导出任务资源（huaweicloud_das_slow_log_export_task）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_slow_log_export_task)
- [Binlog解析任务资源（huaweicloud_das_binlog_parse_task）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_binlog_parse_task)
- [Binlog解析结果导出资源（huaweicloud_das_binlog_parse_task_export）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_binlog_parse_task_export)

### 资源/数据源依赖关系

```text
data.huaweicloud_das_database_users
    └── huaweicloud_das_binlog_parse_task
        └── huaweicloud_das_binlog_parse_task_export

huaweicloud_das_slow_log_export_task
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询数据库用户列表

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建Binlog解析任务：

```hcl
variable "log_analysis_instance_id" {
  description = "The ID of the database instance"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下指定数据库实例的用户列表信息，用于创建Binlog解析任务
data "huaweicloud_das_database_users" "test" {
  instance_id = var.log_analysis_instance_id
}

locals {
  user_id = try(data.huaweicloud_das_database_users.test.users[0].id, null)
}
```

**参数说明**：
- **instance_id**：数据库实例ID，通过引用输入变量 `log_analysis_instance_id` 进行赋值

### 3. 创建慢日志导出任务

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建慢日志导出任务资源：

```hcl
variable "slow_log_bucket_name" {
  description = "The OBS bucket name for exporting slow logs"
  type        = string
}

variable "slow_log_start_time" {
  description = "The start time of the slow logs to export, in RFC3339 format"
  type        = string
}

variable "slow_log_end_time" {
  description = "The end time of the slow logs to export, in RFC3339 format"
  type        = string
}

variable "slow_log_file_path" {
  description = "The OBS file directory for the export task"
  type        = string
  default     = null
}

variable "slow_log_export_type" {
  description = "The export type for the slow log export task"
  type        = string
  default     = null
}

variable "slow_log_sort_field" {
  description = "The sort field for the slow log export task"
  type        = string
  default     = null
}

variable "slow_log_sort_asc" {
  description = "Whether to sort in ascending order"
  type        = bool
  default     = null
}

variable "slow_log_time_zone" {
  description = "The time zone for the slow log export task"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建慢日志导出任务资源
resource "huaweicloud_das_slow_log_export_task" "test" {
  instance_id = var.log_analysis_instance_id
  bucket_name = var.slow_log_bucket_name
  start_time  = var.slow_log_start_time
  end_time    = var.slow_log_end_time
  file_path   = var.slow_log_file_path
  export_type = var.slow_log_export_type
  sort_field  = var.slow_log_sort_field
  sort_asc    = var.slow_log_sort_asc
  time_zone   = var.slow_log_time_zone
}
```

**参数说明**：
- **instance_id**：数据库实例ID，通过引用输入变量 `log_analysis_instance_id` 进行赋值
- **bucket_name**：用于导出慢日志的OBS桶名称，通过引用输入变量 `slow_log_bucket_name` 进行赋值
- **start_time**：待导出慢日志的开始时间（RFC3339格式），通过引用输入变量 `slow_log_start_time` 进行赋值
- **end_time**：待导出慢日志的结束时间（RFC3339格式），通过引用输入变量 `slow_log_end_time` 进行赋值
- **file_path**：导出任务的OBS文件目录，通过引用输入变量 `slow_log_file_path` 进行赋值
- **export_type**：慢日志导出任务的导出类型，通过引用输入变量 `slow_log_export_type` 进行赋值
- **sort_field**：慢日志导出任务的排序字段，通过引用输入变量 `slow_log_sort_field` 进行赋值
- **sort_asc**：是否按升序排序，通过引用输入变量 `slow_log_sort_asc` 进行赋值
- **time_zone**：慢日志导出任务的时区，通过引用输入变量 `slow_log_time_zone` 进行赋值

> 注意：请确保OBS桶已存在，并具备相应的写入权限。

### 4. 创建Binlog解析任务

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建Binlog解析任务资源：

```hcl
variable "binlog_binlog_type" {
  description = "The binlog type"
  type        = string
}

variable "binlog_file_name" {
  description = "The binlog file name"
  type        = string
}

variable "binlog_backup_id" {
  description = "The backup ID"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Binlog解析任务资源
resource "huaweicloud_das_binlog_parse_task" "test" {
  user_id     = local.user_id
  binlog_type = var.binlog_binlog_type
  file_name   = var.binlog_file_name
  backup_id   = var.binlog_backup_id
}
```

**参数说明**：
- **user_id**：数据库用户ID，根据数据库用户列表查询数据源（data.huaweicloud_das_database_users）的返回结果进行赋值
- **binlog_type**：Binlog类型，通过引用输入变量 `binlog_binlog_type` 进行赋值
- **file_name**：Binlog文件名称，通过引用输入变量 `binlog_file_name` 进行赋值
- **backup_id**：备份ID，通过引用输入变量 `binlog_backup_id` 进行赋值

### 5. 导出Binlog解析结果

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建Binlog解析结果导出资源：

```hcl
variable "binlog_export_bucket_name" {
  description = "The OBS bucket name for exporting binlog parse results"
  type        = string
}

variable "binlog_filter_db_names" {
  description = "The list of database names to filter"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "binlog_filter_tb_names" {
  description = "The list of table names to filter"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "binlog_filter_start_time" {
  description = "The start time of the export range, in RFC3339 format"
  type        = string
  default     = null
}

variable "binlog_filter_end_time" {
  description = "The end time of the export range, in RFC3339 format"
  type        = string
  default     = null
}

variable "binlog_filter_types" {
  description = "The list of SQL types to filter"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "binlog_filter_parse_double_insert" {
  description = "Whether to export UPDATE statements as two INSERT statements"
  type        = bool
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Binlog解析结果导出资源
resource "huaweicloud_das_binlog_parse_task_export" "test" {
  user_id     = local.user_id
  task_id     = huaweicloud_das_binlog_parse_task.test.id
  bucket_name = var.binlog_export_bucket_name

  filter_condition {
    db_names            = var.binlog_filter_db_names
    tb_names            = var.binlog_filter_tb_names
    start_time          = var.binlog_filter_start_time
    end_time            = var.binlog_filter_end_time
    types               = var.binlog_filter_types
    parse_double_insert = var.binlog_filter_parse_double_insert
  }
}
```

**参数说明**：
- **user_id**：数据库用户ID，根据数据库用户列表查询数据源（data.huaweicloud_das_database_users）的返回结果进行赋值
- **task_id**：Binlog解析任务ID，通过引用Binlog解析任务资源的ID进行赋值
- **bucket_name**：用于导出Binlog解析结果的OBS桶名称，通过引用输入变量 `binlog_export_bucket_name` 进行赋值
- **filter_condition**：导出过滤条件配置
  - **db_names**：待过滤的数据库名称列表，通过引用输入变量 `binlog_filter_db_names` 进行赋值
  - **tb_names**：待过滤的表名称列表，通过引用输入变量 `binlog_filter_tb_names` 进行赋值
  - **start_time**：导出范围的开始时间（RFC3339格式），通过引用输入变量 `binlog_filter_start_time` 进行赋值
  - **end_time**：导出范围的结束时间（RFC3339格式），通过引用输入变量 `binlog_filter_end_time` 进行赋值
  - **types**：待过滤的SQL类型列表，通过引用输入变量 `binlog_filter_types` 进行赋值
  - **parse_double_insert**：是否将UPDATE语句导出为两条INSERT语句，通过引用输入变量 `binlog_filter_parse_double_insert` 进行赋值

> 注意：Binlog解析结果导出依赖已创建的Binlog解析任务。请确保OBS桶已存在，并具备相应的写入权限。

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 日志分析实例配置
log_analysis_instance_id = "your_rds_instance_id"

# 慢日志导出任务配置
slow_log_bucket_name = "your_obs_bucket"
slow_log_start_time  = "2026-08-12T00:00:00Z"
slow_log_end_time    = "2026-08-13T23:59:59Z"

# Binlog解析任务配置
binlog_binlog_type = "mysql"
binlog_file_name   = "mysql binlog file name"

# Binlog解析结果导出配置
binlog_export_bucket_name         = "your_obs_bucket"
binlog_filter_start_time          = "2000-06-01T00:00:00+08:00"
binlog_filter_end_time            = "2099-06-02T00:00:00+08:00"
binlog_filter_types               = ["insert", "update", "delete", "ddl"]
binlog_filter_parse_double_insert = true
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="log_analysis_instance_id=your_rds_instance_id" -var="slow_log_bucket_name=your_obs_bucket"`
2. 环境变量：`export TF_VAR_log_analysis_instance_id=your_rds_instance_id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建日志分析相关配置
4. 运行 `terraform show` 查看已创建的日志分析相关配置

## 参考信息

- [华为云DAS产品文档](https://support.huaweicloud.com/das/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DAS日志分析最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/das/log-analysis)
