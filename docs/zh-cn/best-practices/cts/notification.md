# 部署通知

## 应用场景

云审计服务（Cloud Trace Service，CTS）的通知功能是一种基于事件驱动的告警机制，可以监控云资源操作并发送实时通知。通过CTS通知，您可以实现安全监控、异常告警、操作审计、实时通知等功能。

CTS通知特别适用于需要实时监控云资源操作、进行安全告警、实现自动化响应等场景，如安全事件监控、异常操作告警、合规性检查、运维自动化等。本最佳实践将介绍如何使用Terraform自动化部署一个CTS通知配置，实现云资源操作的实时监控和通知。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

本最佳实践未使用数据源。

### 资源

- [SMN主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [CTS通知资源（huaweicloud_cts_notification）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cts_notification)

### 资源/数据源依赖关系

```
huaweicloud_smn_topic
    └── huaweicloud_cts_notification
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../docs/introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建SMN主题

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建SMN主题资源：

```hcl
# Variable definitions for SMN resources
variable "topic_name" {
  description = "The name of the SMN topic"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN主题资源
resource "huaweicloud_smn_topic" "test" {
  name = var.topic_name
}
```

**参数说明**：
- **name**：SMN主题的名称，通过引用输入变量topic_name进行赋值

### 3. 创建CTS通知

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CTS通知资源：

```hcl
# Variable definitions for CTS resources
variable "notification_name" {
  description = "The name of the CTS notification"
  type        = string
}

variable "notification_operation_type" {
  description = "The type of operation"
  type        = string
  default     = "customized"
}

variable "notification_agency_name" {
  description = "The name of the IAM agency which allows CTS to access the SMN resources"
  type        = string
}

variable "notification_filter" {
  description = "The filter of the notification"
  type        = list(object({
    condition = string
    rule      = list(string)
  }))
  default  = []
  nullable = false
}

variable "notification_operations" {
  description = "The operations of the notification"
  type        = list(object({
    service     = string
    resource    = string
    trace_names = list(string)
  }))
  default  = []
  nullable = false
}

variable "notification_operation_users" {
  description = "The operation users of the notification"
  type        = list(object({
    group = string
    users = list(string)
  }))
  default  = []
  nullable = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CTS通知资源
resource "huaweicloud_cts_notification" "test" {
  name           = var.notification_name
  operation_type = var.notification_operation_type
  smn_topic      = huaweicloud_smn_topic.test.id
  agency_name    = var.notification_agency_name

  dynamic "filter" {
    for_each = var.notification_filter

    content {
      condition = filter.value.condition
      rule      = filter.value.rule
    }
  }

  dynamic "operations" {
    for_each = var.notification_operations

    content {
      service     = operations.value.service
      resource    = operations.value.resource
      trace_names = operations.value.trace_names
    }
  }

  dynamic "operation_users" {
    for_each = var.notification_operation_users

    content {
      group = operation_users.value.group
      users = operation_users.value.users
    }
  }
}
```

**参数说明**：
- **name**：CTS通知的名称，通过引用输入变量notification_name进行赋值
- **operation_type**：操作类型，通过引用输入变量notification_operation_type进行赋值，默认值为"customized"表示自定义操作
- **smn_topic**：关联的SMN主题ID，通过引用huaweicloud_smn_topic.test.id进行赋值
- **agency_name**：允许CTS访问SMN资源的IAM委托名称，通过引用输入变量notification_agency_name进行赋值
- **filter**：通知过滤器，使用dynamic块动态创建，包含以下参数：
  - **condition**：过滤条件，支持"AND"、"OR"等逻辑操作符
  - **rule**：过滤规则列表，支持基于状态码、资源名称、API版本等条件过滤
- **operations**：监控的操作，使用dynamic块动态创建，包含以下参数：
  - **service**：云服务名称，如"ECS"、"VPC"等
  - **resource**：资源类型，如"ecs"、"vpc"等
  - **trace_names**：跟踪的操作名称列表，如"createServer"、"deleteServer"等
- **operation_users**：操作用户，使用dynamic块动态创建，包含以下参数：
  - **group**：用户组名称
  - **users**：用户名称列表

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# SMN主题配置
topic_name = "tf_test_topic"

# CTS通知配置
notification_name        = "tf_test_notification"
notification_agency_name = "cts_admin_trust"

# 通知过滤器配置
notification_filter = [
  {
    condition = "OR"
    rule      = [
      "code = 400",
      "resource_name = name",
      "api_version = 1.0",
    ]
  }
]

# 监控操作配置
notification_operations = [
  {
    service     = "ECS"
    resource    = "ecs"
    trace_names = [
      "createServer",
      "deleteServer",
    ]
  }
]

# 操作用户配置
notification_operation_users = [
  {
    group = "devops"
    users = [
      "your_operation_user_name",
    ]
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="notification_name=my-notification" -var="topic_name=my-topic"`
2. 环境变量：`export TF_VAR_notification_name=my-notification`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建SMN主题和CTS通知
4. 运行 `terraform show` 查看已创建的SMN主题和CTS通知

## 参考信息

- [华为云CTS产品文档](https://support.huaweicloud.com/cts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CTS通知最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cts/notification)
