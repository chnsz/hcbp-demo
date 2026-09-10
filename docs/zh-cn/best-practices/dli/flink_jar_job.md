# 部署Flink Jar作业

## 应用场景

数据湖探索（Data Lake Insight，DLI）是华为云提供的大数据计算与分析服务，支持SQL、Flink和Spark等多种计算引擎，帮助用户快速构建数据湖分析业务。本实践将指导您使用Terraform在华为云上创建一个DLI弹性资源池、一个通用队列，并部署一个Flink Jar作业，实现流式数据处理。

本最佳实践将介绍如何使用Terraform自动化部署DLI弹性资源池、通用队列和Flink Jar作业，包括资源池的CIDR配置、队列的CU数量设置以及作业的JAR包路径、Flink版本等参数的配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DLI弹性资源池（huaweicloud_dli_elastic_resource_pool）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_elastic_resource_pool)
- [DLI队列（huaweicloud_dli_queue）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_queue)
- [DLI Flink Jar作业（huaweicloud_dli_flinkjar_job）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_flinkjar_job)

### 资源/数据源依赖关系

```
huaweicloud_dli_elastic_resource_pool
    └── huaweicloud_dli_queue
        └── huaweicloud_dli_flinkjar_job
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建弹性资源池

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性资源池
variable "elastic_resource_pool_name" {
  description = "弹性资源池的名称"
  type        = string
}

variable "elastic_resource_pool_cidr" {
  description = "弹性资源池的CIDR地址段"
  type        = string
}

variable "elastic_resource_pool_description" {
  description = "弹性资源池的描述信息"
  type        = string
  default     = ""
}

variable "elastic_resource_pool_min_cu" {
  description = "弹性资源池的最小CU数"
  type        = number
  default     = 16
}

variable "elastic_resource_pool_max_cu" {
  description = "弹性资源池的最大CU数"
  type        = number
  default     = 64
}

variable "enterprise_project_id" {
  description = "企业项目ID"
  type        = string
  default     = ""
}

variable "elastic_resource_pool_label" {
  description = "弹性资源池的标签"
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
- **name**：通过引用输入变量 elastic_resource_pool_name 进行赋值，用于指定弹性资源池的名称。
- **description**：通过引用输入变量 elastic_resource_pool_description 进行赋值，用于描述弹性资源池。
- **min_cu**：通过引用输入变量 elastic_resource_pool_min_cu 进行赋值，用于指定弹性资源池的最小CU数。
- **max_cu**：通过引用输入变量 elastic_resource_pool_max_cu 进行赋值，用于指定弹性资源池的最大CU数。
- **cidr**：通过引用输入变量 elastic_resource_pool_cidr 进行赋值，用于指定弹性资源池的CIDR地址段。
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，当变量值不为空时设置企业项目ID。
- **label**：通过引用输入变量 elastic_resource_pool_label 进行赋值，用于给弹性资源池添加标签。

### 3. 创建通用队列

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建通用队列
variable "queue_name" {
  description = "DLI通用队列的名称"
  type        = string
}

variable "queue_cu_count" {
  description = "DLI队列的CU数量"
  type        = number
  default     = 16
}

variable "queue_description" {
  description = "DLI队列的描述信息"
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
- **elastic_resource_pool_name**：通过引用已创建的弹性资源池的名称进行赋值，将队列关联到该资源池。
- **resource_mode**：固定为1，表示队列类型为专属资源池模式。
- **name**：通过引用输入变量 queue_name 进行赋值，用于指定队列的名称。
- **queue_type**：固定为“general”，表示队列类型为通用队列。
- **cu_count**：通过引用输入变量 queue_cu_count 进行赋值，用于指定队列的CU数量。
- **description**：通过引用输入变量 queue_description 进行赋值，用于描述队列。
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，当变量值不为空时设置企业项目ID。

### 4. 创建Flink Jar作业

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Flink Jar作业
variable "job_name" {
  description = "Flink Jar作业的名称"
  type        = string
}

variable "job_description" {
  description = "Flink Jar作业的描述信息"
  type        = string
  default     = ""
}

variable "job_main_class" {
  description = "Flink Jar作业的主类"
  type        = string
  default     = ""
}

variable "job_entrypoint" {
  description = "JAR包所在的OBS路径"
  type        = string
}

variable "job_entrypoint_args" {
  description = "Flink Jar作业的入口参数"
  type        = string
  default     = ""
}

variable "job_dependency_jars" {
  description = "Flink Jar作业的依赖JAR包列表"
  type        = list(string)
  default     = []
}

variable "job_dependency_files" {
  description = "Flink Jar作业的依赖文件列表"
  type        = list(string)
  default     = []
}

variable "job_execution_agency_urn" {
  description = "授权给DLI的委托URN"
  type        = string
  default     = ""
}

variable "job_feature" {
  description = "Flink镜像的特性类型，支持basic和custom"
  type        = string
  default     = "basic"
}

variable "job_flink_version" {
  description = "Flink版本"
  type        = string
}

variable "job_image" {
  description = "自定义Flink镜像，当feature为custom时生效"
  type        = string
  default     = ""
}

variable "job_cu_num" {
  description = "Flink Jar作业所需的CU数"
  type        = number
  default     = 2
}

variable "job_parallel_num" {
  description = "Flink Jar作业的并行度"
  type        = number
  default     = 1
}

variable "job_obs_bucket" {
  description = "用于保存作业日志的OBS桶"
  type        = string
  default     = ""
}

variable "job_log_enabled" {
  description = "是否开启将作业日志上传到OBS桶的功能"
  type        = bool
  default     = false
}

variable "job_smn_topic" {
  description = "用于接收作业失败通知的SMN主题"
  type        = string
  default     = ""
}

variable "job_restart_when_exception" {
  description = "作业异常时是否自动重启"
  type        = bool
  default     = false
}

variable "job_manager_cu_num" {
  description = "JobManager的CU数量"
  type        = number
  default     = 1
}

variable "job_tm_cu_num" {
  description = "每个Task Manager的CU数量"
  type        = number
  default     = 1
}

variable "job_tm_slot_num" {
  description = "每个Task Manager的槽位数"
  type        = number
  default     = null
}

variable "job_resume_checkpoint" {
  description = "异常时是否从检查点恢复"
  type        = bool
  default     = false
}

variable "job_resume_max_num" {
  description = "异常时的最大重试次数"
  type        = number
  default     = -1
}

variable "job_runtime_config" {
  description = "Flink作业运行时的自定义优化参数"
  type        = map(string)
  default     = {}
}

variable "job_checkpoint_path" {
  description = "用于存储检查点的OBS路径"
  type        = string
  default     = ""
}

variable "job_tags" {
  description = "Flink Jar作业的标签"
  type        = map(string)
  default     = {}
}

variable "job_checkpoint_enabled" {
  description = "是否开启自动作业快照功能"
  type        = bool
  default     = false
}

variable "job_checkpoint_mode" {
  description = "快照模式"
  type        = number
  default     = 1
}

variable "job_checkpoint_interval" {
  description = "快照间隔（秒）"
  type        = number
  default     = 30
}

resource "huaweicloud_dli_flinkjar_job" "test" {
  name                   = var.job_name
  description            = var.job_description
  queue_name             = huaweicloud_dli_queue.test.name
  main_class             = var.job_main_class
  entrypoint             = var.job_entrypoint
  entrypoint_args        = var.job_entrypoint_args
  dependency_jars        = var.job_dependency_jars
  dependency_files       = var.job_dependency_files
  execution_agency_urn   = var.job_execution_agency_urn
  feature                = var.job_feature
  flink_version          = var.job_flink_version
  image                  = var.job_image
  cu_num                 = var.job_cu_num
  parallel_num           = var.job_parallel_num
  obs_bucket             = var.job_obs_bucket
  log_enabled            = var.job_log_enabled
  smn_topic              = var.job_smn_topic
  restart_when_exception = var.job_restart_when_exception
  manager_cu_num         = var.job_manager_cu_num
  tm_cu_num              = var.job_tm_cu_num
  tm_slot_num            = var.job_tm_slot_num
  resume_checkpoint      = var.job_resume_checkpoint
  resume_max_num         = var.job_resume_max_num
  runtime_config         = var.job_runtime_config
  checkpoint_path        = var.job_checkpoint_path
  tags                   = var.job_tags
  checkpoint_enabled     = var.job_checkpoint_enabled
  checkpoint_mode        = var.job_checkpoint_mode
  checkpoint_interval    = var.job_checkpoint_interval

  depends_on = [huaweicloud_dli_queue.test]
}
```

**参数说明**：
- **name**：通过引用输入变量 job_name 进行赋值，用于指定Flink Jar作业的名称。
- **description**：通过引用输入变量 job_description 进行赋值，用于描述作业。
- **queue_name**：通过引用已创建的DLI队列的名称进行赋值，指定作业运行的队列。
- **main_class**：通过引用输入变量 job_main_class 进行赋值，指定作业的主类。
- **entrypoint**：通过引用输入变量 job_entrypoint 进行赋值，指定JAR包所在的OBS路径。
- **entrypoint_args**：通过引用输入变量 job_entrypoint_args 进行赋值，指定作业的入口参数。
- **dependency_jars**：通过引用输入变量 job_dependency_jars 进行赋值，指定依赖的JAR包列表。
- **dependency_files**：通过引用输入变量 job_dependency_files 进行赋值，指定依赖的文件列表。
- **execution_agency_urn**：通过引用输入变量 job_execution_agency_urn 进行赋值，指定授权给DLI的委托URN。
- **feature**：通过引用输入变量 job_feature 进行赋值，指定Flink镜像的特性类型。
- **flink_version**：通过引用输入变量 job_flink_version 进行赋值，指定Flink版本。
- **image**：通过引用输入变量 job_image 进行赋值，指定自定义Flink镜像。
- **cu_num**：通过引用输入变量 job_cu_num 进行赋值，指定作业所需的CU数。
- **parallel_num**：通过引用输入变量 job_parallel_num 进行赋值，指定作业的并行度。
- **obs_bucket**：通过引用输入变量 job_obs_bucket 进行赋值，指定用于保存日志的OBS桶。
- **log_enabled**：通过引用输入变量 job_log_enabled 进行赋值，指定是否开启日志上传。
- **smn_topic**：通过引用输入变量 job_smn_topic 进行赋值，指定接收失败通知的SMN主题。
- **restart_when_exception**：通过引用输入变量 job_restart_when_exception 进行赋值，指定异常时是否自动重启。
- **manager_cu_num**：通过引用输入变量 job_manager_cu_num 进行赋值，指定JobManager的CU数量。
- **tm_cu_num**：通过引用输入变量 job_tm_cu_num 进行赋值，指定每个Task Manager的CU数量。
- **tm_slot_num**：通过引用输入变量 job_tm_slot_num 进行赋值，指定每个Task Manager的槽位数。
- **resume_checkpoint**：通过引用输入变量 job_resume_checkpoint 进行赋值，指定异常时是否从检查点恢复。
- **resume_max_num**：通过引用输入变量 job_resume_max_num 进行赋值，指定异常时的最大重试次数。
- **runtime_config**：通过引用输入变量 job_runtime_config 进行赋值，指定运行时的自定义优化参数。
- **checkpoint_path**：通过引用输入变量 job_checkpoint_path 进行赋值，指定存储检查点的OBS路径。
- **tags**：通过引用输入变量 job_tags 进行赋值，用于给作业添加标签。
- **checkpoint_enabled**：通过引用输入变量 job_checkpoint_enabled 进行赋值，指定是否开启自动快照。
- **checkpoint_mode**：通过引用输入变量 job_checkpoint_mode 进行赋值，指定快照模式。
- **checkpoint_interval**：通过引用输入变量 job_checkpoint_interval 进行赋值，指定快照间隔。

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
elastic_resource_pool_name = "your_elastic_resource_pool_name"
elastic_resource_pool_cidr = "172.16.0.0/18"
queue_name                 = "your_general_queue_name"
job_name                   = "your_flink_jar_job_name"
job_entrypoint             = "obs://your_bucket_path/your.jar"
job_flink_version          = "1.15"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Flink Jar作业
4. 运行 `terraform show` 查看已创建的Flink Jar作业

## 参考信息

- [华为云数据湖探索产品文档](https://support.huaweicloud.com/dli/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DLI Flink Jar作业最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dli/flink-jar-job)
