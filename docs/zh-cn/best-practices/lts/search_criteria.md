# 部署搜索条件

## 应用场景

云日志服务（Log Tank Service，LTS）是华为云提供的一站式日志管理服务，支持日志的采集、存储、查询与分析。在实际运维中，用户经常需要对同一日志流反复执行相同的检索语句，例如按关键字过滤原始日志或按可视化分析语句统计指标。搜索条件（Search Criteria）可以将这些常用的检索语句保存下来，便于在控制台中快速复用，避免重复编写查询语句。

本最佳实践将介绍如何使用Terraform自动化部署LTS搜索条件，包括日志组、日志流和搜索条件的创建，支持原始日志（ORIGINALLOG）与可视化（VISUALIZATION）两种检索类型，并可通过标签和日志过期时间对日志资源进行统一管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [日志组（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [日志流（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [搜索条件（huaweicloud_lts_search_criteria）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_search_criteria)

### 资源/数据源依赖关系

```
huaweicloud_lts_group
    └── huaweicloud_lts_stream
            └── huaweicloud_lts_search_criteria
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建日志组

在TF文件（如main.tf）中添加以下脚本以创建日志组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建日志组资源
variable "group_name" {
  description = "The name of the log group"
  type        = string
}

variable "group_log_expiration_days" {
  description = "The log expiration days of the log group"
  type        = number
  default     = 14
}

variable "group_tags" {
  description = "The tags of the log group"
  type        = map(string)
  default     = {}
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the log group and log stream"
  type        = string
  default     = null
}

resource "huaweicloud_lts_group" "test" {
  group_name            = var.group_name
  ttl_in_days           = var.group_log_expiration_days
  tags                  = var.group_tags
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **group_name**：日志组名称，通过引用输入变量 group_name 进行赋值
- **ttl_in_days**：日志在日志组中的保存天数，通过引用输入变量 group_log_expiration_days 进行赋值，默认值为14天
- **tags**：日志组的标签，通过引用输入变量 group_tags 进行赋值，可用于资源分类和成本分摊
- **enterprise_project_id**：日志组所属的企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值，用于企业项目隔离

### 3. 创建日志流

在TF文件（如main.tf）中添加以下脚本以创建日志流：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建日志流资源
variable "stream_name" {
  description = "The name of the log stream"
  type        = string
}

variable "stream_log_expiration_days" {
  description = "The log expiration days of the log stream"
  type        = number
  default     = null
}

variable "stream_tags" {
  description = "The tags of the log stream"
  type        = map(string)
  default     = {}
}

variable "stream_is_favorite" {
  description = "Whether to favorite the log stream"
  type        = bool
  default     = false
}

resource "huaweicloud_lts_stream" "test" {
  group_id              = huaweicloud_lts_group.test.id
  stream_name           = var.stream_name
  ttl_in_days           = var.stream_log_expiration_days
  tags                  = var.stream_tags
  enterprise_project_id = var.enterprise_project_id
  is_favorite           = var.stream_is_favorite
}
```

**参数说明**：
- **group_id**：日志流所属的日志组ID，引用上一步创建的日志组 huaweicloud_lts_group.test.id 进行赋值
- **stream_name**：日志流名称，通过引用输入变量 stream_name 进行赋值
- **ttl_in_days**：日志在日志流中的保存天数，通过引用输入变量 stream_log_expiration_days 进行赋值；取值为 null 或 -1 时表示与日志组的保存天数保持一致
- **tags**：日志流的标签，通过引用输入变量 stream_tags 进行赋值
- **enterprise_project_id**：日志流所属的企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **is_favorite**：是否收藏该日志流，通过引用输入变量 stream_is_favorite 进行赋值

### 4. 创建搜索条件

在TF文件（如main.tf）中添加以下脚本以创建搜索条件：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建搜索条件资源
variable "search_criteria" {
  description = "The content of the search criteria"
  type        = string
}

variable "search_criteria_name" {
  description = "The name of the search criteria"
  type        = string
}

variable "search_criteria_type" {
  description = "The type of the search criteria. Available types are ORIGINALLOG and VISUALIZATION"
  type        = string
  default     = "ORIGINALLOG"
}

resource "huaweicloud_lts_search_criteria" "test" {
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id

  criteria = var.search_criteria
  name     = var.search_criteria_name
  type     = var.search_criteria_type
}
```

**参数说明**：
- **log_group_id**：搜索条件所属的日志组ID，引用前面创建的日志组 huaweicloud_lts_group.test.id 进行赋值
- **log_stream_id**：搜索条件所属的日志流ID，引用前面创建的日志流 huaweicloud_lts_stream.test.id 进行赋值
- **criteria**：搜索条件的内容，通过引用输入变量 search_criteria 进行赋值
- **name**：搜索条件的名称，通过引用输入变量 search_criteria_name 进行赋值，仅支持英文字母、数字、中文、连字符、下划线和句点，且不能以句点或下划线开头、不能以句点结尾
- **type**：搜索条件的类型，通过引用输入变量 search_criteria_type 进行赋值，可选值为 ORIGINALLOG（原始日志）和 VISUALIZATION（可视化）

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# LTS资源变量
group_name           = "tf_test_log_group"
stream_name          = "tf_test_log_stream"
search_criteria      = "content:test"
search_criteria_name = "tf_test_search_criteria"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="group_name=my-group"`
2. 环境变量：`export TF_VAR_group_name=my-group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建搜索条件
4. 运行 `terraform show` 查看已创建的搜索条件

## 参考信息

- [华为云云日志服务产品文档](https://support.huaweicloud.com/lts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [LTS搜索条件最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/lts/search-criteria)
