# 部署日志流

## 应用场景

华为云云日志服务（LTS）的日志流功能是日志管理的基础组件，用于组织和存储日志数据。通过创建日志组和日志流，您可以实现日志的分类管理、生命周期控制和标签分类等功能。日志流支持多种日志采集方式，能够接收来自不同应用和服务的日志数据，并提供强大的查询和分析能力。

本最佳实践特别适用于需要集中管理应用日志、实现日志分类存储、构建日志监控和分析系统的场景，如应用运维监控、安全审计、业务数据分析等。本最佳实践将介绍如何使用Terraform自动化部署LTS日志流，包括日志组和日志流的创建，实现完整的日志管理基础设施。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [日志组资源（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [日志流资源（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)

### 资源/数据源依赖关系

```
huaweicloud_lts_group
    └── huaweicloud_lts_stream
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

variable "group_tags" {
  description = "The tags of the log group"
  type        = map(string)
  default     = {}
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the log group"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建日志组资源
resource "huaweicloud_lts_group" "test" {
  group_name            = var.group_name
  ttl_in_days           = var.group_log_expiration_days
  tags                  = var.group_tags
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **group_name**：日志组的名称，通过引用输入变量group_name进行赋值
- **ttl_in_days**：日志组的日志过期天数，通过引用输入变量group_log_expiration_days进行赋值，默认值为14
- **tags**：日志组的标签，通过引用输入变量group_tags进行赋值，默认值为空映射
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建日志流资源
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
- **group_id**：日志流所属的日志组ID，引用前面创建的日志组资源的ID
- **stream_name**：日志流的名称，通过引用输入变量stream_name进行赋值
- **ttl_in_days**：日志流的日志过期天数，通过引用输入变量stream_log_expiration_days进行赋值，默认值为null（继承日志组设置）
- **tags**：日志流的标签，通过引用输入变量stream_tags进行赋值，默认值为空映射
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null
- **is_favorite**：是否收藏日志流，通过引用输入变量stream_is_favorite进行赋值，默认值为false

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 日志组配置
group_name = "tf_test_server_group"

# 日志流配置
stream_name = "tf_test_stream"
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

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建日志流
4. 运行 `terraform show` 查看已创建的日志流详情

## 参考信息

- [华为云云日志服务产品文档](https://support.huaweicloud.com/lts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [LTS日志流最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/lts/log-stream)
