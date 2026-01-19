# 部署主题与AOM告警通知

## 应用场景

消息通知服务（Simple Message Notification，SMN）是华为云提供的可靠、可扩展的消息通知服务，支持多种消息通知方式，包括邮件、短信、HTTP/HTTPS等。应用运维管理（Application Operations Management，AOM）是华为云提供的一站式应用运维管理平台，支持应用监控、日志管理和告警管理等功能。通过将AOM告警通知配置到SMN主题，可以实现告警消息的自动推送。同时，通过配置SMN日志桶，可以将SMN主题的操作日志存储到云日志服务（Log Tank Service，LTS），实现日志的统一管理和分析。本最佳实践将介绍如何使用Terraform自动化部署主题与AOM告警通知，包括创建SMN主题、LTS日志组和日志流、SMN日志桶配置和AOM告警动作规则配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [SMN主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [LTS日志组资源（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS日志流资源（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [SMN日志桶资源（huaweicloud_smn_logtank）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_logtank)
- [AOM告警动作规则资源（huaweicloud_aom_alarm_action_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_alarm_action_rule)

### 资源/数据源依赖关系

```text
huaweicloud_smn_topic
    ├── huaweicloud_smn_logtank
    └── huaweicloud_aom_alarm_action_rule

huaweicloud_lts_group
    └── huaweicloud_lts_stream
        └── huaweicloud_smn_logtank
```

> 注意：SMN日志桶需要依赖SMN主题和LTS日志流，用于将SMN主题的操作日志存储到LTS。AOM告警动作规则需要依赖SMN主题，用于配置告警通知的发送目标。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建SMN主题

在TF文件（如main.tf）中添加以下脚本以创建SMN主题：

```hcl
variable "smn_topic_name" {
  description = "The name of the SMN topic used to send AOM alarm notifications"
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

### 3. 创建LTS日志组和日志流

在TF文件（如main.tf）中添加以下脚本以创建LTS日志组和日志流：

```hcl
variable "lts_group_name" {
  description = "The name of the LTS group"
  type        = string
}

variable "lts_group_ttl_in_days" {
  description = "The TTL in days of the LTS group"
  type        = number
  default     = 30
}

variable "lts_stream_name" {
  description = "The name of the LTS stream"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建LTS日志组资源
resource "huaweicloud_lts_group" "test" {
  group_name            = var.lts_group_name
  ttl_in_days           = var.lts_group_ttl_in_days
  enterprise_project_id = var.enterprise_project_id
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建LTS日志流资源
resource "huaweicloud_lts_stream" "test" {
  group_id              = huaweicloud_lts_group.test.id
  stream_name           = var.lts_stream_name
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **group_name**：日志组名称，通过引用输入变量 `lts_group_name` 进行赋值
- **ttl_in_days**：日志组TTL（天），通过引用输入变量 `lts_group_ttl_in_days` 进行赋值，默认为30天
- **stream_name**：日志流名称，通过引用输入变量 `lts_stream_name` 进行赋值
- **group_id**：日志组ID，通过引用LTS日志组资源的ID进行赋值

> 注意：日志流必须属于某个日志组，因此需要先创建日志组。日志组的TTL用于设置日志的保留时间。

### 4. 配置SMN日志桶

在TF文件（如main.tf）中添加以下脚本以配置SMN日志桶：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN日志桶资源
resource "huaweicloud_smn_logtank" "test" {
  topic_urn     = huaweicloud_smn_topic.test.topic_urn
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id
}
```

**参数说明**：
- **topic_urn**：主题URN，通过引用SMN主题资源的 `topic_urn` 进行赋值
- **log_group_id**：日志组ID，通过引用LTS日志组资源的ID进行赋值
- **log_stream_id**：日志流ID，通过引用LTS日志流资源的ID进行赋值

> 注意：SMN日志桶用于将SMN主题的操作日志存储到LTS，实现日志的统一管理和分析。

### 5. 配置AOM告警动作规则

在TF文件（如main.tf）中添加以下脚本以配置AOM告警动作规则：

```hcl
variable "alarm_action_rule_name" {
  description = "The name of the AOM alarm action rule that used to send the SMN notification"
  type        = string
}

variable "alarm_action_rule_user_name" {
  description = "The user name of the AOM alarm action rule"
  type        = string
}

variable "alarm_action_rule_type" {
  description = "The type of the AOM alarm action rule"
  type        = string
  default     = "1"
}

variable "alarm_action_rule_description" {
  description = "The description of the AOM alarm action rule"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AOM告警动作规则资源
resource "huaweicloud_aom_alarm_action_rule" "test" {
  name                  = var.alarm_action_rule_name
  user_name             = var.alarm_action_rule_user_name
  type                  = var.alarm_action_rule_type
  notification_template = "aom.built-in.template.zh"
  description           = var.alarm_action_rule_description

  smn_topics {
    topic_urn = huaweicloud_smn_topic.test.topic_urn
  }
}
```

**参数说明**：
- **name**：告警动作规则名称，通过引用输入变量 `alarm_action_rule_name` 进行赋值
- **user_name**：用户名，通过引用输入变量 `alarm_action_rule_user_name` 进行赋值
- **type**：规则类型，通过引用输入变量 `alarm_action_rule_type` 进行赋值，默认为 `1`（通知类型）
- **notification_template**：通知模板，设置为 `aom.built-in.template.zh` 表示使用AOM内置的中文模板
- **description**：规则描述，通过引用输入变量 `alarm_action_rule_description` 进行赋值，可选参数
- **smn_topics**：SMN主题配置，通过引用SMN主题资源的 `topic_urn` 进行赋值，用于指定告警通知的发送目标

> 注意：AOM告警动作规则用于配置告警通知的发送方式，通过配置SMN主题可以实现告警消息的自动推送。通知模板可以选择内置模板或自定义模板。

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# SMN主题配置（必填）
smn_topic_name = "tf_test_topic"

# LTS日志组和日志流配置（必填）
lts_group_name  = "tf_test_group"
lts_stream_name = "tf_test_stream"

# AOM告警动作规则配置（必填）
alarm_action_rule_name      = "tf_test_action_rule"
alarm_action_rule_user_name = "your_operation_user_name"
alarm_action_rule_type      = "1"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="smn_topic_name=tf_test_topic" -var="lts_group_name=tf_test_group"`
2. 环境变量：`export TF_VAR_smn_topic_name=tf_test_topic`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建SMN主题、LTS日志组和日志流、SMN日志桶和AOM告警动作规则
4. 运行 `terraform show` 查看已创建的资源

## 参考信息

- [华为云SMN产品文档](https://support.huaweicloud.com/smn/index.html)
- [华为云AOM产品文档](https://support.huaweicloud.com/aom/index.html)
- [华为云LTS产品文档](https://support.huaweicloud.com/lts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SMN主题与AOM告警通知最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/smn/topic-with-aom-alarm-notification)
