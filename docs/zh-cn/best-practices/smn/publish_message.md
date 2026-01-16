# 部署消息发布

## 应用场景

消息通知服务（Simple Message Notification，SMN）是华为云提供的可靠、可扩展的消息通知服务，支持多种消息通知方式，包括邮件、短信、HTTP/HTTPS等。通过消息发布功能，可以向SMN主题发布消息，订阅该主题的终端将收到消息通知。消息发布支持直接消息内容、消息结构体和消息模板等多种方式，满足不同场景的消息发布需求。本最佳实践将介绍如何使用Terraform自动化部署消息发布，包括创建SMN主题、订阅、消息模板（可选）和发布消息。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [SMN主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [SMN订阅资源（huaweicloud_smn_subscription）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_subscription)
- [SMN消息模板资源（huaweicloud_smn_message_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_message_template)
- [SMN消息发布资源（huaweicloud_smn_message_publish）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_message_publish)

### 资源/数据源依赖关系

```text
huaweicloud_smn_topic
    ├── huaweicloud_smn_subscription
    └── huaweicloud_smn_message_publish

huaweicloud_smn_message_template
    └── huaweicloud_smn_message_publish
```

> 注意：消息发布需要依赖SMN主题，可以选择使用消息模板。订阅用于接收消息通知，消息发布后订阅该主题的终端将收到消息。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建SMN主题

在TF文件（如main.tf）中添加以下脚本以创建SMN主题：

```hcl
variable "topic_name" {
  description = "The name of the SMN topic"
  type        = string
}

variable "topic_display_name" {
  description = "The display name of the SMN topic"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN主题资源
resource "huaweicloud_smn_topic" "test" {
  name                  = var.topic_name
  display_name          = var.topic_display_name
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **name**：主题名称，通过引用输入变量 `topic_name` 进行赋值
- **display_name**：主题显示名称，通过引用输入变量 `topic_display_name` 进行赋值，可选参数
- **enterprise_project_id**：企业项目ID，通过引用输入变量 `enterprise_project_id` 进行赋值，可选参数

### 3. 创建SMN订阅

在TF文件（如main.tf）中添加以下脚本以创建SMN订阅：

```hcl
variable "subscription_protocol" {
  description = "The protocol of the subscription"
  type        = string
}

variable "subscription_endpoint" {
  description = "The endpoint of the subscription"
  type        = string
}

variable "subscription_description" {
  description = "The remark for SMN subscriptions"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN订阅资源
resource "huaweicloud_smn_subscription" "test" {
  topic_urn = huaweicloud_smn_topic.test.id
  protocol  = var.subscription_protocol
  endpoint  = var.subscription_endpoint
  remark    = var.subscription_description
}
```

**参数说明**：
- **topic_urn**：主题URN，通过引用SMN主题资源的ID进行赋值
- **protocol**：订阅协议，通过引用输入变量 `subscription_protocol` 进行赋值，支持 `email`、`sms`、`http`、`https`、`functionstage` 等
- **endpoint**：订阅终端，通过引用输入变量 `subscription_endpoint` 进行赋值，根据协议类型填写对应的终端地址
- **remark**：订阅备注，通过引用输入变量 `subscription_description` 进行赋值，可选参数

> 注意：订阅协议和终端需要匹配，例如短信协议需要填写手机号码，邮件协议需要填写邮箱地址。

### 4. 创建消息模板（可选）

在TF文件（如main.tf）中添加以下脚本以创建消息模板（可选）：

```hcl
variable "template_name" {
  description = "The name of the message template"
  type        = string
  default     = ""
  nullable    = false
}

variable "template_protocol" {
  description = "The protocol of the message template"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.template_name == "" || var.template_protocol != ""
    error_message = "The template_protocol is required if template_name is provided."
  }
}

variable "template_content" {
  description = "The content of the message template"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.template_name == "" || var.template_content != ""
    error_message = "The template_content is required if template_name is provided."
  }
}

# 创建消息模板（当template_name不为空时创建）
resource "huaweicloud_smn_message_template" "test" {
  count    = var.template_name != "" ? 1 : 0

  name     = var.template_name
  protocol = var.template_protocol
  content  = var.template_content
}
```

**参数说明**：
- **count**：资源数量，当 `template_name` 不为空时创建资源
- **name**：模板名称，通过引用输入变量 `template_name` 进行赋值
- **protocol**：模板协议，通过引用输入变量 `template_protocol` 进行赋值
- **content**：模板内容，通过引用输入变量 `template_content` 进行赋值

> 注意：消息模板是可选的，如果使用模板发布消息，需要先创建消息模板。模板协议需要与订阅协议匹配。

### 5. 发布消息

在TF文件（如main.tf）中添加以下脚本以发布消息：

```hcl
variable "pulblish_subject" {
  description = "The subject of the message"
  type        = string
}

variable "pulblish_message" {
  description = "The message content (mutually exclusive with message_structure)"
  type        = string
  default     = ""
  nullable    = false
}

variable "pulblish_message_structure" {
  description = "The JSON message structure that allows sending different content to different protocol subscribers (mutually exclusive with message)"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = !(var.template_name == "" && var.pulblish_message == "") || var.pulblish_message_structure != ""
    error_message = "The pulblish_message_structure is required if both template_name and pulblish_message are not provided."
  }
}

variable "pulblish_time_to_live" {
  description = "The maximum retention time of the message within the SMN system in seconds (default: 3600, max: 86400)"
  type        = string
  default     = null
}

variable "pulblish_tags" {
  description = "The tags of the message"
  type        = map(string)
  default     = {}
}

variable "pulblish_message_attributes" {
  description = "The message attributes of the message"
  type        = list(object({
    name   = string
    type   = string
    value  = optional(string)
    values = optional(list(string))
  }))

  default  = []
  nullable = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下发布消息
resource "huaweicloud_smn_message_publish" "test" {
  topic_urn             = huaweicloud_smn_topic.test.topic_urn
  subject               = var.pulblish_subject
  message               = var.pulblish_message != "" ? var.pulblish_message : null
  message_structure     = var.pulblish_message_structure != "" ? var.pulblish_message_structure : null
  message_template_name = var.template_name != "" ? try(huaweicloud_smn_message_template.test[0].name, null) : null
  time_to_live          = var.pulblish_time_to_live
  tags                  = var.pulblish_tags

  dynamic "message_attributes" {
    for_each = var.pulblish_message_attributes

    content {
      name   = message_attributes.value.name
      type   = message_attributes.value.type
      value  = message_attributes.value.value
      values = message_attributes.value.values
    }
  }
}
```

**参数说明**：
- **topic_urn**：主题URN，通过引用SMN主题资源的 `topic_urn` 进行赋值
- **subject**：消息主题，通过引用输入变量 `pulblish_subject` 进行赋值
- **message**：消息内容，通过引用输入变量 `pulblish_message` 进行赋值，与 `message_structure` 互斥，可选参数
- **message_structure**：消息结构体，通过引用输入变量 `pulblish_message_structure` 进行赋值，JSON格式，允许向不同协议订阅者发送不同内容，与 `message` 互斥，可选参数
- **message_template_name**：消息模板名称，当 `template_name` 不为空时引用消息模板资源的名称，可选参数
- **time_to_live**：消息在SMN系统中的最大保留时间（秒），通过引用输入变量 `pulblish_time_to_live` 进行赋值，默认为3600秒，最大86400秒，可选参数
- **tags**：消息标签，通过引用输入变量 `pulblish_tags` 进行赋值，可选参数
- **message_attributes**：消息属性列表，通过动态块 `dynamic "message_attributes"` 根据输入变量 `pulblish_message_attributes` 创建多个消息属性，可选参数

> 注意：消息发布支持三种方式：直接消息内容（message）、消息结构体（message_structure）和消息模板（message_template_name）。消息和消息结构体互斥，如果使用消息模板，则不需要提供消息内容。消息结构体为JSON格式，可以为不同协议的订阅者发送不同的内容。

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# SMN主题配置（必填）
topic_name = "tf_test_topic"

# SMN订阅配置（必填）
subscription_protocol      = "sms"
subscription_endpoint      = "18629199536"

# 消息发布配置（必填）
pulblish_subject           = "tf_test_subject"
pulblish_message_structure = "{\"default\":\"Dear user, this is a default message.\",\"sms\":\"Dear user, this is an SMS message.\"}"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="topic_name=tf_test_topic" -var="subscription_protocol=sms"`
2. 环境变量：`export TF_VAR_topic_name=tf_test_topic`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建SMN主题、订阅和发布消息
4. 运行 `terraform show` 查看已创建的资源

## 参考信息

- [华为云SMN产品文档](https://support.huaweicloud.com/smn/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SMN消息发布最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/smn/publish-message)
