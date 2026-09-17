# 部署任务LTS配置与对比策略

## 应用场景

数据复制服务（Data Replication Service，DRS）是华为云提供的一站式数据复制服务，支持数据库上云、数据库迁移、数据库实时同步和数据库灾备等场景。在实时同步或迁移任务运行过程中，用户往往需要将任务日志投递到云日志服务（LTS）进行集中检索与分析，同时通过周期性的数据对比策略及时发现源端与目标端的数据差异，保障数据一致性。

本最佳实践将介绍如何使用Terraform为已存在的DRS任务配置LTS日志投递，并开启周期性的数据对比策略。实践首先创建LTS日志组与日志流，随后开启DRS任务的LTS日志投递，最后为同一任务开启数据对比策略，帮助您以基础设施即代码（IaC）的方式高效管理DRS任务的日志与数据校验能力。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [云日志组（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [云日志流（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [DRS任务LTS配置（huaweicloud_drs_lts_config）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_lts_config)
- [DRS任务对比策略（huaweicloud_drs_compare_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_compare_policy)

### 资源/数据源依赖关系

```
huaweicloud_lts_group
    └── huaweicloud_lts_stream
            └── huaweicloud_drs_lts_config
                    └── huaweicloud_drs_compare_policy
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建LTS日志组

在TF文件（如main.tf）中添加以下脚本以创建用于存储DRS任务日志的LTS日志组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建LTS日志组资源
variable "lts_group_name" {
  description = "The name of the LTS group used to store the DRS job logs"
  type        = string
}

resource "huaweicloud_lts_group" "test" {
  group_name  = var.lts_group_name
  ttl_in_days = 30
}
```

**参数说明**：
- **group_name**：日志组名称，通过引用输入变量 lts_group_name 进行赋值
- **ttl_in_days**：日志存储时长（单位：天），此处设置为30天

### 3. 创建LTS日志流

在TF文件（如main.tf）中添加以下脚本以在日志组下创建日志流：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建LTS日志流资源
variable "lts_stream_name" {
  description = "The name of the LTS stream used to store the DRS job logs"
  type        = string
}

resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.lts_stream_name
}
```

**参数说明**：
- **group_id**：日志流所属的日志组ID，引用上一步创建的日志组资源 huaweicloud_lts_group.test 的ID进行赋值
- **stream_name**：日志流名称，通过引用输入变量 lts_stream_name 进行赋值

### 4. 配置DRS任务的LTS日志投递

在TF文件（如main.tf）中添加以下脚本以开启DRS任务的LTS日志投递：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DRS任务LTS配置资源
variable "drs_job_id" {
  description = "The ID of the existing DRS job"
  type        = string
}

resource "huaweicloud_drs_lts_config" "test" {
  job_id        = var.drs_job_id
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id
}
```

**参数说明**：
- **job_id**：已存在的DRS任务ID，通过引用输入变量 drs_job_id 进行赋值
- **log_group_id**：日志组ID，引用前面创建的日志组资源 huaweicloud_lts_group.test 的ID进行赋值
- **log_stream_id**：日志流ID，引用前面创建的日志流资源 huaweicloud_lts_stream.test 的ID进行赋值

### 5. 配置DRS任务的数据对比策略

在TF文件（如main.tf）中添加以下脚本以开启DRS任务的周期性数据对比策略：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DRS任务对比策略资源
variable "compare_policy_period" {
  description = "The comparison period of the compare policy, e.g. * * 1,3,5 for weekly comparison"
  type        = string
}

variable "compare_policy_begin_time" {
  description = "The start time when the comparison policy takes effect, UTC time in HH:mm:ss format"
  type        = string
}

variable "compare_policy_end_time" {
  description = "The end time when the comparison policy takes effect, UTC time in HH:mm:ss format"
  type        = string
}

variable "compare_policy_compare_type" {
  description = "The list of comparison types, valid values are object_comparison, lines and account"
  type        = list(string)
  default     = ["lines"]
}

variable "compare_policy_compare_policy" {
  description = "The comparison policy, valid values are normal and manyToOne"
  type        = string
  default     = "normal"
}

variable "compare_policy_interval_hour" {
  description = "The comparison interval in hours, required for hourly comparison"
  type        = number
  default     = null
}

resource "huaweicloud_drs_compare_policy" "test" {
  job_id         = var.drs_job_id
  period         = var.compare_policy_period
  begin_time     = var.compare_policy_begin_time
  end_time       = var.compare_policy_end_time
  compare_type   = var.compare_policy_compare_type
  compare_policy = var.compare_policy_compare_policy
  interval_hour  = var.compare_policy_interval_hour

  depends_on = [huaweicloud_drs_lts_config.test]
}
```

**参数说明**：
- **job_id**：已存在的DRS任务ID，通过引用输入变量 drs_job_id 进行赋值
- **period**：对比周期，通过引用输入变量 compare_policy_period 进行赋值，例如 `* * 1,3,5` 表示每周一、三、五执行对比
- **begin_time**：对比策略生效的开始时间（UTC时间，格式为 HH:mm:ss），通过引用输入变量 compare_policy_begin_time 进行赋值
- **end_time**：对比策略生效的结束时间（UTC时间，格式为 HH:mm:ss），通过引用输入变量 compare_policy_end_time 进行赋值
- **compare_type**：对比类型列表，通过引用输入变量 compare_policy_compare_type 进行赋值，可选值为 object_comparison、lines 和 account
- **compare_policy**：对比策略，通过引用输入变量 compare_policy_compare_policy 进行赋值，可选值为 normal 和 manyToOne
- **interval_hour**：按小时对比时的对比间隔（单位：小时），通过引用输入变量 compare_policy_interval_hour 进行赋值
- **depends_on**：显式声明对DRS任务LTS配置资源的依赖，确保先完成LTS日志投递配置再开启对比策略

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 资源变量
lts_group_name            = "tf-test-drs-lts-group"
lts_stream_name           = "tf-test-drs-lts-stream"
drs_job_id                = "your-drs-job-id"
compare_policy_period     = "* * 1,3,5"
compare_policy_begin_time = "00:00:00"
compare_policy_end_time   = "04:00:00"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="drs_job_id=your-drs-job-id"`
2. 环境变量：`export TF_VAR_drs_job_id=your-drs-job-id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始配置DRS任务的LTS日志投递与数据对比策略
4. 运行 `terraform show` 查看已配置的DRS任务LTS日志投递与数据对比策略

## 参考信息

- [华为云数据复制服务产品文档](https://support.huaweicloud.com/drs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DRS任务LTS配置与对比策略最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/job-lts-config-with-compare-policy)
