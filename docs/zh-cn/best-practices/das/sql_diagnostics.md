# 部署SQL诊断

## 应用场景

数据管理服务（Data Admin Service，简称DAS）提供SQL诊断与智能运维能力，支持对数据库实例批量开启SQL采集开关，并对数据库连接启用搜索路径开关，帮助DBA分析慢SQL、全量SQL等运行数据，提升问题定位效率。本最佳实践将介绍如何使用Terraform自动化部署DAS SQL诊断相关配置，包括批量设置SQL开关，以及为数据库连接开启搜索路径开关。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [批量设置SQL开关资源（huaweicloud_das_batch_set_sql_switch）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_batch_set_sql_switch)
- [搜索路径开关资源（huaweicloud_das_search_path_switch）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_search_path_switch)

### 资源/数据源依赖关系

```text
huaweicloud_das_batch_set_sql_switch

huaweicloud_das_search_path_switch
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 批量设置SQL开关

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建批量设置SQL开关资源：

```hcl
variable "sql_diagnostics_engine_type" {
  description = "The engine type of the instances"
  type        = string
}

variable "batch_sql_switch_on" {
  description = "Whether to enable the SQL switch"
  type        = bool
}

variable "batch_sql_switch_type" {
  description = "The type of SQL switch to set"
  type        = string
}

variable "batch_sql_instance_ids" {
  description = "The list of instance IDs"
  type        = list(string)
}

variable "batch_sql_retention_hours" {
  description = "The retention hours of the SQL data"
  type        = number
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建批量设置SQL开关资源
resource "huaweicloud_das_batch_set_sql_switch" "test" {
  engine_type     = var.sql_diagnostics_engine_type
  switch_on       = var.batch_sql_switch_on
  switch_type     = var.batch_sql_switch_type
  instance_ids    = var.batch_sql_instance_ids
  retention_hours = var.batch_sql_retention_hours
}
```

**参数说明**：
- **engine_type**：实例的引擎类型，通过引用输入变量 `sql_diagnostics_engine_type` 进行赋值
- **switch_on**：是否开启SQL开关，通过引用输入变量 `batch_sql_switch_on` 进行赋值
- **switch_type**：待设置的SQL开关类型，通过引用输入变量 `batch_sql_switch_type` 进行赋值
- **instance_ids**：实例ID列表，通过引用输入变量 `batch_sql_instance_ids` 进行赋值
- **retention_hours**：SQL数据的保留时长（小时），通过引用输入变量 `batch_sql_retention_hours` 进行赋值

> 注意：部署前需确保目标数据库实例已存在。请根据业务需要合理配置SQL数据保留时长。

### 3. 开启搜索路径开关

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建搜索路径开关资源：

```hcl
variable "search_path_connection_id" {
  description = "The ID of the database connection (DB user ID)"
  type        = string
}

variable "search_path_switch_on" {
  description = "Whether to enable the search path switch"
  type        = bool
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建搜索路径开关资源
resource "huaweicloud_das_search_path_switch" "test" {
  connection_id = var.search_path_connection_id
  switch_on     = var.search_path_switch_on
}
```

**参数说明**：
- **connection_id**：数据库连接ID（数据库用户ID），通过引用输入变量 `search_path_connection_id` 进行赋值
- **switch_on**：是否开启搜索路径开关，通过引用输入变量 `search_path_switch_on` 进行赋值

> 注意：部署前需确保对应的数据库连接（数据库用户）已存在。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 批量设置SQL开关配置
sql_diagnostics_engine_type = "MySQL"
batch_sql_switch_on         = true
batch_sql_switch_type       = "DAS_QUERY"
batch_sql_instance_ids      = ["tf_test_instance_id_1", "tf_test_instance_id_2"]
batch_sql_retention_hours   = 24

# 搜索路径开关配置
search_path_connection_id = "tf_test_connection_id"
search_path_switch_on     = true
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="batch_sql_switch_on=true" -var="search_path_switch_on=true"`
2. 环境变量：`export TF_VAR_sql_diagnostics_engine_type=MySQL`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建SQL诊断相关配置
4. 运行 `terraform show` 查看已创建的SQL诊断相关配置

## 参考信息

- [华为云DAS产品文档](https://support.huaweicloud.com/das/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DAS SQL诊断最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/das/sql-diagnostics)
