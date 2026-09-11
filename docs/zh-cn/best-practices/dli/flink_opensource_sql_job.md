# 部署Flink OpenSource SQL作业

## 应用场景

数据湖探索（Data Lake Insight，DLI）是华为云提供的大数据计算与分析服务，支持SQL、Flink和Spark等多种计算引擎。Flink OpenSource SQL作业允许用户以标准Flink SQL语句描述数据源、查询逻辑与结果输出，无需编写Java/Scala代码即可完成流式数据处理。

本最佳实践将介绍如何使用Terraform自动化部署DLI弹性资源池、通用队列和Flink OpenSource SQL作业，包括弹性资源池创建、队列配置以及作业参数设置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DLI弹性资源池（huaweicloud_dli_elastic_resource_pool）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_elastic_resource_pool)
- [DLI队列（huaweicloud_dli_queue）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_queue)
- [DLI Flink SQL作业（huaweicloud_dli_flinksql_job）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_flinksql_job)

### 资源/数据源依赖关系

```
huaweicloud_dli_elastic_resource_pool
    └── huaweicloud_dli_queue
            └── huaweicloud_dli_flinksql_job
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DLI弹性资源池

在TF文件（如main.tf）中添加以下脚本以创建DLI弹性资源池：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DLI弹性资源池
variable "elastic_resource_pool_name" {
  description = "The name of the DLI elastic resource pool"
  type        = string
}

variable "elastic_resource_pool_description" {
  description = "The description of the elastic resource pool"
  type        = string
  default     = ""
}

variable "elastic_resource_pool_min_cu" {
  description = "The minimum number of CUs for the elastic resource pool"
  type        = number
  default     = 16
}

variable "elastic_resource_pool_max_cu" {
  description = "The maximum number of CUs for the elastic resource pool"
  type        = number
  default     = 64
}

variable "elastic_resource_pool_cidr" {
  description = "The CIDR block of the elastic resource pool."
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
}

variable "elastic_resource_pool_label" {
  description = "The label of the elastic resource pool"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_dli_elastic_resource_pool" "test" {
  name                  = var.elastic_resource_pool_name
  description           = var.elastic_resource_pool_description
  min_cu                = var.elastic_resource_pool_min_cu
  max_cu                = var.elastic_resource_pool_max_cu
  cidr                  = var.elastic_resource_pool_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  label                 = var.elastic_resource_pool_label
}
```

**参数说明**：
- **name**：弹性资源池名称，通过引用输入变量 elastic_resource_pool_name 进行赋值
- **description**：弹性资源池描述，通过引用输入变量 elastic_resource_pool_description 进行赋值
- **min_cu**：弹性资源池的最小CU数，通过引用输入变量 elastic_resource_pool_min_cu 进行赋值
- **max_cu**：弹性资源池的最大CU数，通过引用输入变量 elastic_resource_pool_max_cu 进行赋值
- **cidr**：弹性资源池的网段，通过引用输入变量 elastic_resource_pool_cidr 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值，为空时使用null
- **label**：弹性资源池标签，通过引用输入变量 elastic_resource_pool_label 进行赋值

### 3. 创建DLI队列

在TF文件中添加以下脚本以创建DLI通用队列：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DLI队列
variable "queue_name" {
  description = "The name of the DLI general queue used to run the Flink OpenSource SQL job"
  type        = string
}

variable "queue_cu_count" {
  description = "The CU count of the DLI queue"
  type        = number
  default     = 16
}

variable "queue_description" {
  description = "The description of the DLI queue"
  type        = string
  default     = ""
}

resource "huaweicloud_dli_queue" "test" {
  elastic_resource_pool_name = huaweicloud_dli_elastic_resource_pool.test.name
  resource_mode              = 1
  name                       = var.queue_name
  queue_type                 = "general"
  cu_count                   = var.queue_cu_count
  description                = var.queue_description
  enterprise_project_id      = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **elastic_resource_pool_name**：队列所属的弹性资源池名称，引用上一步创建的弹性资源池名称进行赋值
- **resource_mode**：资源模式，固定为1（弹性资源池模式）
- **name**：队列名称，通过引用输入变量 queue_name 进行赋值
- **queue_type**：队列类型，固定为general（通用队列）
- **cu_count**：队列CU数，通过引用输入变量 queue_cu_count 进行赋值
- **description**：队列描述，通过引用输入变量 queue_description 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值，为空时使用null

### 4. 创建Flink OpenSource SQL作业

在TF文件中添加以下脚本以创建Flink OpenSource SQL作业：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Flink OpenSource SQL作业
variable "job_name" {
  description = "The name of the Flink OpenSource SQL job"
  type        = string
}

variable "job_sql" {
  description = "The Flink SQL statement that includes source, query and sink"
  type        = string
}

variable "job_flink_version" {
  description = "The Flink version"
  type        = string
}

variable "job_execution_agency_urn" {
  description = "The agency URN authorized to DLI"
  type        = string
  default     = ""
}

variable "job_cu_number" {
  description = "The number of CUs selected for the Flink OpenSource SQL job"
  type        = number
  default     = 2
}

variable "job_parallel_number" {
  description = "The parallelism of the Flink OpenSource SQL job"
  type        = number
  default     = 1
}

variable "job_manager_cu_number" {
  description = "The number of CUs in the JobManager"
  type        = number
  default     = 1
}

variable "job_tm_cus" {
  description = "The number of CUs for each Task Manager"
  type        = number
  default     = 1
}

variable "job_tm_slot_num" {
  description = "The number of slots in each Task Manager"
  type        = number
  default     = null
}

variable "job_checkpoint_enabled" {
  description = "Whether to enable the automatic job snapshot function"
  type        = bool
  default     = false
}

variable "job_checkpoint_mode" {
  description = "The snapshot mode."
  type        = string
  default     = "exactly_once"
}

variable "job_checkpoint_interval" {
  description = "The snapshot interval in seconds"
  type        = number
  default     = 10
}

variable "job_obs_bucket" {
  description = "The OBS bucket used to save snapshots or logs"
  type        = string
  default     = ""
}

variable "job_log_enabled" {
  description = "Whether to enable uploading job logs to the OBS bucket"
  type        = bool
  default     = false
}

variable "job_smn_topic" {
  description = "The SMN topic used to receive job failure notifications"
  type        = string
  default     = ""
}

variable "job_restart_when_exception" {
  description = "Whether to enable the function of automatically restarting a job upon job exceptions"
  type        = bool
  default     = false
}

variable "job_resume_max_num" {
  description = "The maximum number of retry times upon exceptions"
  type        = number
  default     = -1
}

variable "job_idle_state_retention" {
  description = "The idle state retention in seconds"
  type        = number
  default     = 1
}

variable "job_runtime_config" {
  description = "The custom optimization parameters when the Flink job is running"
  type        = map(string)
  default     = {}
}

variable "job_tags" {
  description = "The key/value pairs to associate with the Flink OpenSource SQL job"
  type        = map(string)
  default     = {}
}

variable "job_description" {
  description = "The description of the Flink OpenSource SQL job"
  type        = string
  default     = ""
}

resource "huaweicloud_dli_flinksql_job" "test" {
  name                   = var.job_name
  type                   = "flink_opensource_sql_job"
  run_mode               = "exclusive_cluster"
  queue_name             = huaweicloud_dli_queue.test.name
  sql                    = var.job_sql
  flink_version          = var.job_flink_version
  execution_agency_urn   = var.job_execution_agency_urn
  cu_number              = var.job_cu_number
  parallel_number        = var.job_parallel_number
  manager_cu_number      = var.job_manager_cu_number
  tm_cus                 = var.job_tm_cus
  tm_slot_num            = var.job_tm_slot_num
  checkpoint_enabled     = var.job_checkpoint_enabled
  checkpoint_mode        = var.job_checkpoint_mode
  checkpoint_interval    = var.job_checkpoint_interval
  obs_bucket             = var.job_obs_bucket
  log_enabled            = var.job_log_enabled
  smn_topic              = var.job_smn_topic
  restart_when_exception = var.job_restart_when_exception
  resume_max_num         = var.job_resume_max_num
  idle_state_retention   = var.job_idle_state_retention
  runtime_config         = var.job_runtime_config
  tags                   = var.job_tags
  description            = var.job_description

  depends_on = [huaweicloud_dli_queue.test]
}
```

**参数说明**：
- **name**：作业名称，通过引用输入变量 job_name 进行赋值
- **type**：作业类型，固定为flink_opensource_sql_job
- **run_mode**：运行模式，固定为exclusive_cluster（独占集群）
- **queue_name**：作业运行的队列名称，引用上一步创建的队列名称进行赋值
- **sql**：Flink SQL语句，包含数据源、查询和结果输出，通过引用输入变量 job_sql 进行赋值
- **flink_version**：Flink版本，通过引用输入变量 job_flink_version 进行赋值
- **execution_agency_urn**：授权给DLI的委托URN，通过引用输入变量 job_execution_agency_urn 进行赋值
- **cu_number**：作业选择的CU数，通过引用输入变量 job_cu_number 进行赋值
- **parallel_number**：作业并行度，通过引用输入变量 job_parallel_number 进行赋值
- **manager_cu_number**：JobManager的CU数，通过引用输入变量 job_manager_cu_number 进行赋值
- **tm_cus**：每个Task Manager的CU数，通过引用输入变量 job_tm_cus 进行赋值
- **tm_slot_num**：每个Task Manager的slot数，通过引用输入变量 job_tm_slot_num 进行赋值
- **checkpoint_enabled**：是否开启自动快照功能，通过引用输入变量 job_checkpoint_enabled 进行赋值
- **checkpoint_mode**：快照模式，通过引用输入变量 job_checkpoint_mode 进行赋值
- **checkpoint_interval**：快照间隔时间（秒），通过引用输入变量 job_checkpoint_interval 进行赋值
- **obs_bucket**：用于保存快照或日志的OBS桶，通过引用输入变量 job_obs_bucket 进行赋值
- **log_enabled**：是否开启上传作业日志到OBS桶，通过引用输入变量 job_log_enabled 进行赋值
- **smn_topic**：用于接收作业失败通知的SMN主题，通过引用输入变量 job_smn_topic 进行赋值
- **restart_when_exception**：是否开启作业异常自动重启功能，通过引用输入变量 job_restart_when_exception 进行赋值
- **resume_max_num**：异常时最大重试次数，通过引用输入变量 job_resume_max_num 进行赋值
- **idle_state_retention**：空闲状态保留时间（秒），通过引用输入变量 job_idle_state_retention 进行赋值
- **runtime_config**：Flink作业运行时的自定义优化参数，通过引用输入变量 job_runtime_config 进行赋值
- **tags**：作业标签键值对，通过引用输入变量 job_tags 进行赋值
- **description**：作业描述，通过引用输入变量 job_description 进行赋值

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
elastic_resource_pool_name = "tf_test_resource_pool"
elastic_resource_pool_cidr = "172.16.0.0/18"

elastic_resource_pool_label = {
  spec = "basic"
}

queue_name        = "tf_test_queue"
job_name          = "tf_test_flink_sql_job"
job_flink_version = "1.15"
job_sql           = <<-EOF
create table dataGenSource(
  user_id string,
  amount int
) with (
  'connector' = 'datagen',
  'rows-per-second' = '1',
  'fields.user_id.kind' = 'random',
  'fields.user_id.length' = '3'
);

create table printSink(
  user_id string,
  amount int
) with (
  'connector' = 'print'
);

insert into printSink select * from dataGenSource;
EOF
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="elastic_resource_pool_name=my-pool"`
2. 环境变量：`export TF_VAR_elastic_resource_pool_name=my-pool`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Flink OpenSource SQL作业
4. 运行 `terraform show` 查看已创建的Flink OpenSource SQL作业

## 参考信息

- [华为云数据湖探索产品文档](https://support.huaweicloud.com/dli/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DLI Flink OpenSource SQL作业最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dli/flink-opensource-sql-job)
