# 部署CES事件告警规则

## 应用场景

消息通知服务（Simple Message Notification，SMN）是华为云提供的可靠、可扩展的消息通知服务，支持多种消息通知方式，包括邮件、短信、HTTP/HTTPS等。云监控服务（Cloud Eye Service，CES）是华为云提供的监控服务，支持资源的实时监控和告警。通过CES事件告警规则，可以监控SMN主题的事件，当满足告警条件时自动发送通知。本最佳实践将介绍如何使用Terraform自动化部署CES事件告警规则，包括创建SMN主题和配置CES告警规则，实现SMN主题事件的监控和告警。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [SMN主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [CES告警规则资源（huaweicloud_ces_alarmrule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarmrule)

### 资源/数据源依赖关系

```text
huaweicloud_smn_topic
    └── huaweicloud_ces_alarmrule
```

> 注意：CES告警规则需要引用SMN主题的URN来发送告警通知，因此告警规则依赖于SMN主题资源。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建SMN主题

在TF文件（如main.tf）中添加以下脚本以创建SMN主题：

```hcl
variable "smn_topic_name" {
  description = "The name of the SMN topic used to send alarm notifications"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN主题资源
resource "huaweicloud_smn_topic" "test" {
  name                  = var.smn_topic_name
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **name**：主题名称，通过引用输入变量 `smn_topic_name` 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 `enterprise_project_id` 进行赋值，可选参数

### 3. 创建CES告警规则

在TF文件（如main.tf）中添加以下脚本以创建CES告警规则：

```hcl
variable "alarm_rule_name" {
  description = "The name of the CES alarm rule"
  type        = string
}

variable "alarm_rule_description" {
  description = "The description of the CES alarm rule"
  type        = string
  default     = null
}

variable "alarm_action_enabled" {
  description = "Whether to enable the action to be triggered by an alarm"
  type        = bool
  default     = true
}

variable "alarm_enabled" {
  description = "Whether to enable the alarm"
  type        = bool
  default     = true
}

variable "alarm_type" {
  description = "The type of the alarm"
  type        = string
  default     = "ALL_INSTANCE"
}

variable "alarm_rule_conditions" {
  description = "The list of alarm rule conditions"
  type        = list(object({
    metric_name         = string
    period              = string
    filter              = string
    comparison_operator = string
    value               = string
    count               = string
    unit                = optional(string)
    suppress_duration   = optional(string)
    alarm_level         = optional(string)
  }))

  nullable = false
}

variable "alarm_rule_resource" {
  description = "The list of resource dimensions for specified monitoring scope"
  type        = list(object({
    name  = string
    value = optional(string)
  }))

  default  = []
  nullable = true
}

variable "alarm_rule_notification_begin_time" {
  description = "The alarm notification start time"
  type        = string
  default     = null
}

variable "alarm_rule_notification_end_time" {
  description = "The alarm notification stop time"
  type        = string
  default     = null
}

variable "alarm_rule_effective_timezone" {
  description = "The time zone"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CES告警规则资源
resource "huaweicloud_ces_alarmrule" "test" {
  alarm_name            = var.alarm_rule_name
  alarm_description     = var.alarm_rule_description
  alarm_action_enabled  = var.alarm_action_enabled
  alarm_enabled         = var.alarm_enabled
  alarm_type            = var.alarm_type
  enterprise_project_id = var.enterprise_project_id

  metric {
    namespace = "SYS.SMN"
  }

  dynamic "condition" {
    for_each = var.alarm_rule_conditions

    content {
      metric_name         = condition.value.metric_name
      period              = condition.value.period
      filter              = condition.value.filter
      comparison_operator = condition.value.comparison_operator
      value               = condition.value.value
      unit                = condition.value.unit
      count               = condition.value.count
      suppress_duration   = condition.value.suppress_duration
      alarm_level         = condition.value.alarm_level
    }
  }

  dynamic "resources" {
    for_each = var.alarm_rule_resource

    content {
      dimensions {
        name  = resources.value.name
        value = resources.value.value
      }
    }
  }

  alarm_actions {
    type = "notification"

    notification_list = [
      huaweicloud_smn_topic.test.topic_urn
    ]
  }

  notification_begin_time = var.alarm_rule_notification_begin_time
  notification_end_time   = var.alarm_rule_notification_end_time
  effective_timezone      = var.alarm_rule_effective_timezone
}
```

**参数说明**：
- **alarm_name**：告警规则名称，通过引用输入变量 `alarm_rule_name` 进行赋值
- **alarm_description**：告警规则描述，通过引用输入变量 `alarm_rule_description` 进行赋值，可选参数
- **alarm_action_enabled**：是否启用告警动作，通过引用输入变量 `alarm_action_enabled` 进行赋值，默认为 `true`
- **alarm_enabled**：是否启用告警，通过引用输入变量 `alarm_enabled` 进行赋值，默认为 `true`
- **alarm_type**：告警类型，通过引用输入变量 `alarm_type` 进行赋值，默认为 `ALL_INSTANCE`
- **metric**：监控指标配置，命名空间设置为 `SYS.SMN` 表示监控SMN服务
- **condition**：告警条件列表，通过动态块 `dynamic "condition"` 根据输入变量 `alarm_rule_conditions` 创建多个告警条件
- **resources**：监控资源维度列表，通过动态块 `dynamic "resources"` 根据输入变量 `alarm_rule_resource` 创建资源维度
- **alarm_actions**：告警动作配置，类型为 `notification`，通知列表引用SMN主题的URN
- **notification_begin_time**：告警通知开始时间，通过引用输入变量 `alarm_rule_notification_begin_time` 进行赋值，可选参数
- **notification_end_time**：告警通知结束时间，通过引用输入变量 `alarm_rule_notification_end_time` 进行赋值，可选参数
- **effective_timezone**：时区，通过引用输入变量 `alarm_rule_effective_timezone` 进行赋值，可选参数

> 注意：告警规则通过 `alarm_actions` 中的 `notification_list` 引用SMN主题的 `topic_urn`，实现告警通知的发送。告警条件支持多个条件配置，每个条件包含指标名称、周期、过滤方式、比较运算符、阈值等参数。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# SMN主题配置（必填）
smn_topic_name = "tf_test_topic"

# CES告警规则基本信息（必填）
alarm_rule_name        = "tf_test_alarm_rule"
alarm_rule_description = "Monitor SMN topic events"
alarm_type             = "ALL_INSTANCE"

# 告警条件配置（必填）
alarm_rule_conditions = [
  {
    metric_name         = "email_total_count"
    period              = "1"
    filter              = "average"
    comparison_operator = ">="
    value               = "80"
    count               = "3"
    unit                = "count"
    alarm_level         = "3"
  }
]

# 监控资源维度配置（可选）
alarm_rule_resource = [
  {
    name = "topic_id"
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="smn_topic_name=tf_test_topic" -var="alarm_rule_name=tf_test_alarm_rule"`
2. 环境变量：`export TF_VAR_smn_topic_name=tf_test_topic`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建SMN主题和CES告警规则
4. 运行 `terraform show` 查看已创建的资源

## 参考信息

- [华为云SMN产品文档](https://support.huaweicloud.com/smn/index.html)
- [华为云CES产品文档](https://support.huaweicloud.com/ces/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SMN CES事件告警规则最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/smn/ces-event-alarm-rule)
