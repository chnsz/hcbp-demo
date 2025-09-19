# 部署SQL告警规则

## 应用场景

华为云云日志服务（LTS）的SQL告警规则功能允许用户基于SQL查询结果设置告警条件，当查询结果满足预设条件时自动触发告警通知。通过配置SQL告警规则，您可以实现日志数据的实时监控、异常检测和自动告警，提高运维效率和系统可靠性。

本最佳实践特别适用于需要实时监控日志数据、检测系统异常、实现自动化告警通知的场景，如应用性能监控、错误日志告警、业务指标监控、安全事件检测等。本最佳实践将介绍如何使用Terraform自动化部署LTS SQL告警规则，包括SMN主题、日志组、日志流和SQL告警规则的创建，实现完整的日志监控和告警解决方案。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [LTS通知模板查询数据源（data.huaweicloud_lts_notification_templates）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/lts_notification_templates)

### 资源

- [SMN主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [日志组资源（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [日志流资源（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [SQL告警规则资源（huaweicloud_lts_sql_alarm_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_sql_alarm_rule)

### 资源/数据源依赖关系

```
data.huaweicloud_lts_notification_templates
    └── huaweicloud_lts_sql_alarm_rule

huaweicloud_smn_topic
    └── huaweicloud_lts_sql_alarm_rule

huaweicloud_lts_group
    ├── huaweicloud_lts_stream
    └── huaweicloud_lts_sql_alarm_rule

huaweicloud_lts_stream
    └── huaweicloud_lts_sql_alarm_rule
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建SMN主题

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建SMN主题资源：

```hcl
variable "topic_name" {
  description = "The name of the SMN topic"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN主题资源
resource "huaweicloud_smn_topic" "test" {
  name         = var.topic_name
  display_name = "The display name of topic"
}
```

**参数说明**：
- **name**：SMN主题的名称，通过引用输入变量topic_name进行赋值
- **display_name**：SMN主题的显示名称，设置为"The display name of topic"

### 3. 创建日志组

在TF文件中添加以下脚本以告知Terraform创建日志组资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建日志组资源
resource "huaweicloud_lts_group" "test" {
  group_name  = var.group_name
  ttl_in_days = var.group_log_expiration_days
}
```

**参数说明**：
- **group_name**：日志组的名称，通过引用输入变量group_name进行赋值
- **ttl_in_days**：日志组的日志过期天数，通过引用输入变量group_log_expiration_days进行赋值，默认值为14

### 4. 创建日志流

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建日志流资源
resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.stream_name
  ttl_in_days = var.stream_log_expiration_days
}
```

**参数说明**：
- **group_id**：日志流所属的日志组ID，引用前面创建的日志组资源的ID
- **stream_name**：日志流的名称，通过引用输入变量stream_name进行赋值
- **ttl_in_days**：日志流的日志过期天数，通过引用输入变量stream_log_expiration_days进行赋值，默认值为null（继承日志组设置）

### 5. 查询LTS通知模板信息

在TF文件中添加以下脚本以告知Terraform查询LTS通知模板信息：

```hcl
variable "notification_template_name" {
  description = "The name of the notification template"
  type        = string
  default     = ""
  nullable    = false
}

variable "domain_id" {
  description = "The domain ID"
  type        = string
  default     = null
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的LTS通知模板信息，用于创建SQL告警规则
data "huaweicloud_lts_notification_templates" "test" {
  count = var.notification_template_name != "" ? 0 : 1

  domain_id = var.domain_id
}
```

**参数说明**：
- **count**：条件创建，当notification_template_name变量不为空字符串时创建此数据源
- **domain_id**：域ID，通过引用输入变量domain_id进行赋值，默认值为null

### 6. 创建SQL告警规则

在TF文件中添加以下脚本以告知Terraform创建SQL告警规则资源：

```hcl
variable "alarm_rule_name" {
  description = "The name of the SQL alarm rule"
  type        = string
}

variable "alarm_rule_condition_expression" {
  description = "The condition expression of the SQL alarm rule"
  type        = string
}

variable "alarm_rule_alarm_level" {
  description = "The alarm level of the SQL alarm rule"
  type        = string
  default     = "MINOR"
}

variable "alarm_rule_trigger_condition_count" {
  description = "The trigger condition count of the SQL alarm rule"
  type        = number
  default     = 2
}

variable "alarm_rule_trigger_condition_frequency" {
  description = "The trigger condition frequency of the SQL alarm rule"
  type        = number
  default     = 3
}

variable "alarm_rule_send_recovery_notifications" {
  description = "The send recovery notifications of the SQL alarm rule"
  type        = bool
  default     = true
}

variable "alarm_rule_recovery_frequency" {
  description = "The recovery frequency of the SQL alarm rule"
  type        = number
  default     = 4
}

variable "alarm_rule_notification_frequency" {
  description = "The notification frequency of the SQL alarm rule"
  type        = number
  default     = 15
}

variable "alarm_rule_alias" {
  description = "The alias of the SQL alarm rule"
  type        = string
  default     = ""
}

variable "alarm_rule_request_title" {
  description = "The request title of the SQL alarm rule"
  type        = string
}

variable "alarm_rule_request_sql" {
  description = "The request SQL of the SQL alarm rule"
  type        = string
}

variable "alarm_rule_request_search_time_range_unit" {
  description = "The request search time range unit of the SQL alarm rule"
  type        = string
  default     = "minute"
}

variable "alarm_rule_request_search_time_range" {
  description = "The request search time range of the SQL alarm rule"
  type        = number
  default     = 5
}

variable "alarm_rule_frequency_type" {
  description = "The frequency type of the SQL alarm rule"
  type        = string
  default     = "HOURLY"
}

variable "alarm_rule_notification_user_name" {
  description = "The notification user name of the SQL alarm rule"
  type        = string
}

variable "alarm_rule_notification_language" {
  description = "The notification language of the SQL alarm rule"
  type        = string
  default     = "en-us"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SQL告警规则资源
resource "huaweicloud_lts_sql_alarm_rule" "test" {
  name                        = var.alarm_rule_name
  condition_expression        = var.alarm_rule_condition_expression
  alarm_level                 = var.alarm_rule_alarm_level
  send_notifications          = true
  trigger_condition_count     = var.alarm_rule_trigger_condition_count
  trigger_condition_frequency = var.alarm_rule_trigger_condition_frequency
  send_recovery_notifications = var.alarm_rule_send_recovery_notifications
  recovery_frequency          = var.alarm_rule_send_recovery_notifications ? var.alarm_rule_recovery_frequency : null
  notification_frequency      = var.alarm_rule_notification_frequency
  alarm_rule_alias            = var.alarm_rule_alias

  sql_requests {
    title                  = var.alarm_rule_request_title
    sql                    = var.alarm_rule_request_sql
    log_group_id           = huaweicloud_lts_group.test.id
    log_stream_id          = huaweicloud_lts_stream.test.id
    search_time_range_unit = var.alarm_rule_request_search_time_range_unit
    search_time_range      = var.alarm_rule_request_search_time_range
    log_group_name         = huaweicloud_lts_group.test.group_name
    log_stream_name        = huaweicloud_lts_stream.test.stream_name
  }

  frequency {
    type = var.alarm_rule_frequency_type
  }

  notification_save_rule {
    template_name = var.notification_template_name!= "" ? var.notification_template_name : try([for v in data.huaweicloud_lts_notification_templates.test.templates[*].name :v if v == "sql_template"][0], null)
    user_name     = var.alarm_rule_notification_user_name
    language      = var.alarm_rule_notification_language

    topics {
      name         = huaweicloud_smn_topic.test.name
      topic_urn    = huaweicloud_smn_topic.test.topic_urn
      display_name = huaweicloud_smn_topic.test.display_name
      push_policy  = huaweicloud_smn_topic.test.push_policy
    }
  }
}
```

**参数说明**：
- **name**：SQL告警规则的名称，通过引用输入变量alarm_rule_name进行赋值
- **condition_expression**：告警条件表达式，通过引用输入变量alarm_rule_condition_expression进行赋值
- **alarm_level**：告警级别，通过引用输入变量alarm_rule_alarm_level进行赋值，默认值为"MINOR"
- **send_notifications**：是否发送通知，设置为true表示启用通知
- **trigger_condition_count**：触发条件计数，通过引用输入变量alarm_rule_trigger_condition_count进行赋值，默认值为2
- **trigger_condition_frequency**：触发条件频率，通过引用输入变量alarm_rule_trigger_condition_frequency进行赋值，默认值为3
- **send_recovery_notifications**：是否发送恢复通知，通过引用输入变量alarm_rule_send_recovery_notifications进行赋值，默认值为true
- **recovery_frequency**：恢复频率，当send_recovery_notifications为true时使用alarm_rule_recovery_frequency的值
- **notification_frequency**：通知频率，通过引用输入变量alarm_rule_notification_frequency进行赋值，默认值为15
- **alarm_rule_alias**：告警规则别名，通过引用输入变量alarm_rule_alias进行赋值，默认值为空字符串
- **sql_requests**：SQL请求配置块
  - **title**：请求标题，通过引用输入变量alarm_rule_request_title进行赋值
  - **sql**：SQL查询语句，通过引用输入变量alarm_rule_request_sql进行赋值
  - **log_group_id**：日志组ID，引用前面创建的日志组资源的ID
  - **log_stream_id**：日志流ID，引用前面创建的日志流资源的ID
  - **search_time_range_unit**：搜索时间范围单位，通过引用输入变量alarm_rule_request_search_time_range_unit进行赋值，默认值为"minute"
  - **search_time_range**：搜索时间范围，通过引用输入变量alarm_rule_request_search_time_range进行赋值，默认值为5
  - **log_group_name**：日志组名称，引用前面创建的日志组资源的名称
  - **log_stream_name**：日志流名称，引用前面创建的日志流资源的名称
- **frequency**：频率配置块
  - **type**：频率类型，通过引用输入变量alarm_rule_frequency_type进行赋值，默认值为"HOURLY"
- **notification_save_rule**：通知保存规则配置块
  - **template_name**：通知模板名称，优先使用notification_template_name变量，如果为空则尝试从查询结果中获取"sql_template"
  - **user_name**：通知用户名，通过引用输入变量alarm_rule_notification_user_name进行赋值
  - **language**：通知语言，通过引用输入变量alarm_rule_notification_language进行赋值，默认值为"en-us"
  - **topics**：主题配置块
    - **name**：主题名称，引用前面创建的SMN主题资源的名称
    - **topic_urn**：主题URN，引用前面创建的SMN主题资源的URN
    - **display_name**：主题显示名称，引用前面创建的SMN主题资源的显示名称
    - **push_policy**：推送策略，引用前面创建的SMN主题资源的推送策略

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# SMN主题配置
topic_name = "tf-test-topic"

# 日志组配置
group_name = "tf_test_log_group"

# 日志流配置
stream_name = "tf_test_log_stream"

# 通知模板配置
domain_id = "your_domain_id"

# SQL告警规则配置
alarm_rule_name                   = "tf-test-sql-alarm-rule"
alarm_rule_condition_expression   = "t>0"
alarm_rule_request_title          = "tf-test-sql-alarm-rule-title"
alarm_rule_request_sql            = "select count(*) as t"
alarm_rule_notification_user_name = "your_notification_user_name"
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

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建SQL告警规则
4. 运行 `terraform show` 查看已创建的SQL告警规则详情

## 参考信息

- [华为云云日志服务产品文档](https://support.huaweicloud.com/lts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [LTS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/lts)
