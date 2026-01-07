# 部署AOM告警动作回调

## 应用场景

应用运维管理（Application Operations Management，AOM）是华为云提供的一站式应用运维管理平台，支持应用监控、日志管理、告警管理等核心功能。通过配置AOM告警动作回调，可以将告警信息通过消息通知服务（SMN）发送到指定的回调URL，实现告警信息的实时通知和处理。告警动作回调可以帮助用户快速响应告警事件，提高运维效率。

本最佳实践将介绍如何使用Terraform自动化部署AOM告警动作回调，包括创建SMN主题和订阅、AOM消息模板，以及配置AOM告警动作规则。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [消息通知服务主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [消息通知服务订阅资源（huaweicloud_smn_subscription）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_subscription)
- [AOM消息模板资源（huaweicloud_aom_message_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_message_template)
- [AOM告警动作规则资源（huaweicloud_aom_alarm_action_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_alarm_action_rule)

### 资源/数据源依赖关系

```
huaweicloud_smn_topic
    └── huaweicloud_smn_subscription

huaweicloud_aom_message_template
    └── huaweicloud_aom_alarm_action_rule

huaweicloud_smn_topic
    └── huaweicloud_aom_alarm_action_rule
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建消息通知服务主题资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建消息通知服务主题资源：

```hcl
variable "smn_topic_name" {
  description = "The name of the SMN topic that used to send the SMN notification"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建消息通知服务主题资源
resource "huaweicloud_smn_topic" "test" {
  name                  = var.smn_topic_name
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **name**：主题名称，通过引用输入变量smn_topic_name进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，当值为空字符串时设置为null

### 3. 创建消息通知服务订阅资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建消息通知服务订阅资源：

```hcl
variable "alarm_callback_urls" {
  description = "The URLs of the alarm callback"
  type        = list(string)

  validation {
    condition     = length(var.alarm_callback_urls) > 0 && alltrue([for url in var.alarm_callback_urls : length(regexall("^http[s]?://", url)) > 0])
    error_message = "The alarm callback URLs must be provided and must start with http:// or https://"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建消息通知服务订阅资源
resource "huaweicloud_smn_subscription" "test" {
  count = length(var.alarm_callback_urls) > 0 ? 1 : 0

  topic_urn = huaweicloud_smn_topic.test.id
  protocol  = length(regexall("^https?://", var.alarm_callback_urls[count.index])) > 0 ? "https" : "http"
  endpoint  = var.alarm_callback_urls[count.index]
}
```

**参数说明**：
- **count**：订阅资源的创建数，用于控制是否创建订阅，仅当alarm_callback_urls列表不为空时创建
- **topic_urn**：主题的URN，引用前面创建的消息通知服务主题资源（huaweicloud_smn_topic.test）的ID
- **protocol**：消息接收终端的协议，根据回调URL的前缀自动判断为http或https
- **endpoint**：消息接收终端地址，通过引用输入变量alarm_callback_urls列表中的元素进行赋值

> 注意：实际使用中，如果alarm_callback_urls包含多个URL，需要为每个URL创建单独的订阅资源。上述示例代码需要根据实际需求进行调整。

### 4. 创建AOM消息模板资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建AOM消息模板资源：

```hcl
variable "alarm_notification_template_name" {
  description = "The name of the AOM alarm notification template that used to send the SMN notification"
  type        = string
}

variable "alarm_notification_template_locale" {
  description = "The locale of the AOM alarm notification template that used to send the SMN notification"
  type        = string
  default     = "en-us"

  validation {
    condition     = contains(["en-us", "zh-cn"], var.alarm_notification_template_locale)
    error_message = "The alarm notification template locale must be 'en-us' or 'zh-cn'"
  }
}

variable "alarm_notification_template_description" {
  description = "The description of the AOM alarm notification template that used to send the SMN notification"
  type        = string
  default     = ""
}

variable "alarm_notification_template_notification_type" {
  description = "The notification type of the AOM alarm notification template that used to send the SMN notification"
  type        = string
  default     = "email"
}

variable "alarm_notification_template_notification_topic" {
  description = "The notification topic of the AOM alarm notification template that used to send the SMN notification"
  type        = string
  default     = "An alert occurred at time $${starts_at}[$${event_severity}_$${event_type}_$${clear_type}]."
}

variable "alarm_notification_template_content" {
  description = "The content of the AOM alarm notification template that used to send the SMN notification"
  type        = string
  default     = <<EOT
Alarm Name: $${event_name_alias};
Alarm ID: $${id};
Notification Rule: $${action_rule};
Trigger Time: $${starts_at};
Trigger Level: $${event_severity};
Alarm Content: $${alarm_info};
Resource Identifier: $${resources_new};
Remediation Suggestion: $${alarm_fix_suggestion_zh};
  EOT
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AOM消息模板资源
resource "huaweicloud_aom_message_template" "test" {
  name                  = var.alarm_notification_template_name
  locale                = var.alarm_notification_template_locale
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  description           = var.alarm_notification_template_description

  templates {
    sub_type = var.alarm_notification_template_notification_type
    topic    = var.alarm_notification_template_notification_topic
    content  = var.alarm_notification_template_content
  }
}
```

**参数说明**：
- **name**：消息模板名称，通过引用输入变量alarm_notification_template_name进行赋值
- **locale**：消息模板语言，通过引用输入变量alarm_notification_template_locale进行赋值，默认值为"en-us"，可选值为"en-us"或"zh-cn"
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，当值为空字符串时设置为null
- **description**：消息模板描述，通过引用输入变量alarm_notification_template_description进行赋值，默认值为空字符串
- **templates.sub_type**：通知类型，通过引用输入变量alarm_notification_template_notification_type进行赋值，默认值为"email"
- **templates.topic**：通知主题模板，通过引用输入变量alarm_notification_template_notification_topic进行赋值，支持变量替换
- **templates.content**：通知内容模板，通过引用输入变量alarm_notification_template_content进行赋值，支持变量替换

### 5. 创建AOM告警动作规则资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建AOM告警动作规则资源：

```hcl
variable "alarm_action_rule_name" {
  description = "The name of the AOM alarm action rule that used to send the SMN notification"
  type        = string
}

variable "alarm_action_rule_user_name" {
  description = "The user name of the AOM alarm action rule that used to send the SMN notification"
  type        = string
}

variable "alarm_action_rule_type" {
  description = "The type of the AOM alarm action rule that used to send the SMN notification"
  type        = string
  default     = "1" # notification
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AOM告警动作规则资源
resource "huaweicloud_aom_alarm_action_rule" "test" {
  depends_on = [huaweicloud_aom_message_template.test]

  name                  = var.alarm_action_rule_name
  user_name             = var.alarm_action_rule_user_name
  type                  = var.alarm_action_rule_type
  notification_template = huaweicloud_aom_message_template.test.name

  smn_topics {
    topic_urn = huaweicloud_smn_topic.test.topic_urn
  }
}
```

**参数说明**：
- **depends_on**：显式依赖关系，确保AOM消息模板资源先于告警动作规则资源创建
- **name**：告警动作规则名称，通过引用输入变量alarm_action_rule_name进行赋值
- **user_name**：用户名，通过引用输入变量alarm_action_rule_user_name进行赋值
- **type**：告警动作规则类型，通过引用输入变量alarm_action_rule_type进行赋值，默认值为"1"（表示通知类型）
- **notification_template**：通知模板名称，引用前面创建的AOM消息模板资源（huaweicloud_aom_message_template.test）的名称
- **smn_topics.topic_urn**：SMN主题URN，引用前面创建的消息通知服务主题资源（huaweicloud_smn_topic.test）的topic_urn

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 消息通知服务主题配置
smn_topic_name = "tf_test_alarm_action_callback"

# 告警回调URL配置
alarm_callback_urls = ["https://www.example.com"]

# AOM消息模板配置
alarm_notification_template_name        = "tf_test_alarm_action_callback"
alarm_notification_template_description = "This is a AOM alarm notification template created by Terraform"

# AOM告警动作规则配置
alarm_action_rule_name      = "tf_test_alarm_action_callback"
alarm_action_rule_user_name = "your_operation_user_name"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="smn_topic_name=test-topic" -var="alarm_callback_urls=[\"https://www.example.com\"]"`
2. 环境变量：`export TF_VAR_smn_topic_name=test-topic`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建AOM告警动作回调
4. 运行 `terraform show` 查看已创建的AOM告警动作回调详情

## 参考信息

- [华为云AOM产品文档](https://support.huaweicloud.com/aom/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [AOM告警动作回调最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/aom/action-callback)
