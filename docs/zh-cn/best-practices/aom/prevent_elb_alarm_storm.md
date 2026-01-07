# 部署AOM防止ELB告警风暴

## 应用场景

应用运维管理（Application Operations Management，AOM）是华为云提供的一站式应用运维管理平台，支持应用监控、日志管理、告警管理等核心功能。在监控ELB业务层指标时，可能会产生大量重复或相似的告警，导致告警风暴，影响运维效率。通过配置AOM告警分组规则，可以将相似的告警进行分组合并，减少告警噪音，防止告警风暴，提高告警管理的有效性。

本最佳实践将介绍如何使用Terraform自动化部署AOM防止ELB告警风暴，包括创建LTS日志组和日志流、SMN主题和日志转储、AOM告警动作规则、告警分组规则，以及配置告警规则。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [云日志服务日志组资源（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [云日志服务日志流资源（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [消息通知服务主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [消息通知服务日志转储资源（huaweicloud_smn_logtank）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_logtank)
- [AOM告警动作规则资源（huaweicloud_aom_alarm_action_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_alarm_action_rule)
- [AOM告警分组规则资源（huaweicloud_aom_alarm_group_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_alarm_group_rule)
- [AOM告警规则资源（huaweicloud_aomv4_alarm_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aomv4_alarm_rule)

### 资源/数据源依赖关系

```
huaweicloud_lts_group
    └── huaweicloud_lts_stream
        └── huaweicloud_smn_logtank

huaweicloud_smn_topic
    ├── huaweicloud_smn_logtank
    └── huaweicloud_aom_alarm_action_rule
        └── huaweicloud_aom_alarm_group_rule
            └── huaweicloud_aomv4_alarm_rule
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建云日志服务日志组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云日志服务日志组资源：

```hcl
variable "lts_group_name" {
  description = "The name of the LTS group that used to store the SMN notification logs"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云日志服务日志组资源
resource "huaweicloud_lts_group" "test" {
  group_name            = var.lts_group_name
  ttl_in_days           = 30
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **group_name**：日志组名称，通过引用输入变量lts_group_name进行赋值
- **ttl_in_days**：日志保存时间（单位：天），设置为30天
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，当值为空字符串时设置为null

### 3. 创建云日志服务日志流资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云日志服务日志流资源：

```hcl
variable "lts_stream_name" {
  description = "The name of the LTS stream that used to store the SMN notification logs"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云日志服务日志流资源
resource "huaweicloud_lts_stream" "test" {
  group_id              = huaweicloud_lts_group.test.id
  stream_name           = var.lts_stream_name
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **group_id**：日志组ID，引用前面创建的云日志服务日志组资源（huaweicloud_lts_group.test）的ID
- **stream_name**：日志流名称，通过引用输入变量lts_stream_name进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，当值为空字符串时设置为null

### 4. 创建消息通知服务主题资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建消息通知服务主题资源：

```hcl
variable "smn_topic_name" {
  description = "The name of the SMN topic that used to send the SMN notification"
  type        = string
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

### 5. 创建消息通知服务日志转储资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建消息通知服务日志转储资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建消息通知服务日志转储资源
resource "huaweicloud_smn_logtank" "test" {
  topic_urn     = huaweicloud_smn_topic.test.topic_urn
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id
}
```

**参数说明**：
- **topic_urn**：主题URN，引用前面创建的消息通知服务主题资源（huaweicloud_smn_topic.test）的topic_urn
- **log_group_id**：日志组ID，引用前面创建的云日志服务日志组资源（huaweicloud_lts_group.test）的ID
- **log_stream_id**：日志流ID，引用前面创建的云日志服务日志流资源（huaweicloud_lts_stream.test）的ID

### 6. 创建AOM告警动作规则资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建AOM告警动作规则资源：

```hcl
variable "alarm_action_rule_name" {
  description = "The name of the AOM alarm action rule that used to send the SMN notification"
  type        = string
  default     = "apm"
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AOM告警动作规则资源
resource "huaweicloud_aom_alarm_action_rule" "test" {
  name                  = var.alarm_action_rule_name
  user_name             = var.alarm_action_rule_user_name
  type                  = var.alarm_action_rule_type
  notification_template = "aom.built-in.template.zh"

  smn_topics {
    topic_urn = huaweicloud_smn_topic.test.topic_urn
  }
}
```

**参数说明**：
- **name**：告警动作规则名称，通过引用输入变量alarm_action_rule_name进行赋值，默认值为"apm"
- **user_name**：用户名，通过引用输入变量alarm_action_rule_user_name进行赋值
- **type**：告警动作规则类型，通过引用输入变量alarm_action_rule_type进行赋值，默认值为"1"（表示通知类型）
- **notification_template**：通知模板名称，使用内置模板"aom.built-in.template.zh"
- **smn_topics.topic_urn**：SMN主题URN，引用前面创建的消息通知服务主题资源（huaweicloud_smn_topic.test）的topic_urn

### 7. 创建AOM告警分组规则资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建AOM告警分组规则资源：

```hcl
variable "alarm_group_rule_name" {
  description = "The name of the AOM alarm group rule"
  type        = string
}

variable "alarm_group_rule_group_interval" {
  description = "The group interval of the AOM alarm group rule"
  type        = number
  default     = 60
}

variable "alarm_group_rule_group_repeat_waiting" {
  description = "The group repeat waiting of the AOM alarm group rule"
  type        = number
  default     = 3600
}

variable "alarm_group_rule_group_wait" {
  description = "The group wait of the AOM alarm group rule"
  type        = number
  default     = 15
}

variable "alarm_group_rule_description" {
  description = "The description of the AOM alarm group rule"
  type        = string
  default     = ""
}

variable "alarm_group_rule_condition_matching_rules" {
  description = "The condition matching rules of the AOM alarm group rule"
  type = list(object({
    key     = string
    operate = string
    value   = list(string)
  }))
  default = [
    {
      key     = "event_severity"
      operate = "EXIST"
      value   = ["Critical", "Major"]
    },
    {
      key     = "resource_provider"
      operate = "EQUALS"
      value   = ["AOM"]
    }
  ]
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AOM告警分组规则资源
resource "huaweicloud_aom_alarm_group_rule" "test" {
  depends_on = [huaweicloud_aom_alarm_action_rule.test]

  name                  = var.alarm_group_rule_name
  group_by              = ["resource_provider"]
  group_interval        = var.alarm_group_rule_group_interval
  group_repeat_waiting  = var.alarm_group_rule_group_repeat_waiting
  group_wait            = var.alarm_group_rule_group_wait
  description           = var.alarm_group_rule_description != "" ? var.alarm_group_rule_description : null
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  detail {
    bind_notification_rule_ids = [huaweicloud_aom_alarm_action_rule.test.name]

    dynamic "match" {
      for_each = var.alarm_group_rule_condition_matching_rules

      content {
        key     = match.value.key
        operate = match.value.operate
        value   = match.value.value
      }
    }
  }
}
```

**参数说明**：
- **depends_on**：显式依赖关系，确保AOM告警动作规则资源先于告警分组规则资源创建
- **name**：告警分组规则名称，通过引用输入变量alarm_group_rule_name进行赋值
- **group_by**：分组字段列表，设置为["resource_provider"]（表示按资源提供者进行分组）
- **group_interval**：分组检查间隔（单位：秒），通过引用输入变量alarm_group_rule_group_interval进行赋值，默认值为60秒
- **group_repeat_waiting**：分组重复等待时间（单位：秒），通过引用输入变量alarm_group_rule_group_repeat_waiting进行赋值，默认值为3600秒
- **group_wait**：分组等待时间（单位：秒），通过引用输入变量alarm_group_rule_group_wait进行赋值，默认值为15秒
- **description**：告警分组规则描述，通过引用输入变量alarm_group_rule_description进行赋值，当值为空字符串时设置为null
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，当值为空字符串时设置为null
- **detail.bind_notification_rule_ids**：绑定的通知规则ID列表，引用前面创建的AOM告警动作规则资源（huaweicloud_aom_alarm_action_rule.test）的名称
- **detail.match**：匹配条件列表，通过dynamic块根据输入变量alarm_group_rule_condition_matching_rules动态生成，默认过滤Critical和Major级别的告警以及来自AOM的告警

### 8. 创建AOM告警规则资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建AOM告警规则资源：

```hcl
variable "alarm_rule_name" {
  description = "The name of the AOM alarm rule"
  type        = string
}

variable "prometheus_instance_id" {
  description = "The ID of the Prometheus instance"
  type        = string
  default     = "0"
}

variable "alarm_rule_trigger_conditions" {
  description = "The trigger conditions of the AOM alarm rule"
  type = list(object({
    metric_name             = string
    promql                  = string
    promql_for              = string
    aggregate_type          = optional(string, "by")
    aggregation_type        = string
    aggregation_window      = string
    metric_statistic_method = string
    thresholds              = map(any)
    trigger_type            = string
    trigger_interval        = string
    trigger_times           = string
    query_param             = string
    query_match             = string
  }))
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AOM告警规则资源
resource "huaweicloud_aomv4_alarm_rule" "test" {
  name             = var.alarm_rule_name
  type             = "metric"
  enable           = true
  prom_instance_id = var.prometheus_instance_id

  alarm_notifications {
    notification_enable = true
    notification_type   = "alarm_policy"
    route_group_enable  = true
    route_group_rule    = huaweicloud_aom_alarm_group_rule.test.name
    notify_resolved     = true
    notify_triggered    = true
    notify_frequency    = "-1"
  }

  metric_alarm_spec {
    monitor_type = "all_metric"

    recovery_conditions {
      recovery_timeframe = 1
    }

    dynamic "trigger_conditions" {
      for_each = var.alarm_rule_trigger_conditions

      content {
        metric_query_mode       = "PROM"
        metric_name             = trigger_conditions.value.metric_name
        promql                  = trigger_conditions.value.promql
        promql_for              = trigger_conditions.value.promql_for
        aggregate_type          = trigger_conditions.value.aggregate_type
        aggregation_type        = trigger_conditions.value.aggregation_type
        aggregation_window      = trigger_conditions.value.aggregation_window
        metric_statistic_method = trigger_conditions.value.metric_statistic_method
        thresholds              = trigger_conditions.value.thresholds
        trigger_type            = trigger_conditions.value.trigger_type
        trigger_interval        = trigger_conditions.value.trigger_interval
        trigger_times           = trigger_conditions.value.trigger_times
        query_param             = trigger_conditions.value.query_param
        query_match             = trigger_conditions.value.query_match
      }
    }
  }

  lifecycle {
    ignore_changes = [
      metric_alarm_spec # 如需更新此配置，请使用1.82.3以上版本
    ]
  }
}
```

**参数说明**：
- **name**：告警规则名称，通过引用输入变量alarm_rule_name进行赋值
- **type**：告警规则类型，设置为"metric"（表示指标类型）
- **enable**：是否启用告警规则，设置为true
- **prom_instance_id**：Prometheus实例ID，通过引用输入变量prometheus_instance_id进行赋值，默认值为"0"（表示默认的Prometheus_AOM_Default实例）
- **alarm_notifications.notification_enable**：是否启用通知，设置为true
- **alarm_notifications.notification_type**：通知类型，设置为"alarm_policy"（表示告警策略类型）
- **alarm_notifications.route_group_enable**：是否启用路由分组，设置为true
- **alarm_notifications.route_group_rule**：路由分组规则名称，引用前面创建的AOM告警分组规则资源（huaweicloud_aom_alarm_group_rule.test）的名称
- **alarm_notifications.notify_resolved**：是否通知恢复，设置为true
- **alarm_notifications.notify_triggered**：是否通知触发，设置为true
- **alarm_notifications.notify_frequency**：通知频率，设置为"-1"（表示使用告警分组规则的频率设置）
- **metric_alarm_spec.monitor_type**：监控类型，设置为"all_metric"（表示所有指标）
- **metric_alarm_spec.recovery_conditions.recovery_timeframe**：恢复时间范围，设置为1（单位：分钟）
- **metric_alarm_spec.trigger_conditions**：触发条件列表，通过dynamic块根据输入变量alarm_rule_trigger_conditions动态生成

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 云日志服务配置
lts_group_name  = "tf_test_aom_prevent_elb_alarm_storm"
lts_stream_name = "tf_test_aom_prevent_elb_alarm_storm"

# 消息通知服务主题配置
smn_topic_name = "tf_test_aom_prevent_elb_alarm_storm"

# AOM告警动作规则配置
alarm_action_rule_user_name = "servicestage"

# AOM告警分组规则配置
alarm_group_rule_name = "tf_test_aom_prevent_elb_alarm_storm"

# AOM告警规则配置
alarm_rule_name        = "tf_test_aom_prevent_elb_alarm_storm"
prometheus_instance_id = "0"

alarm_rule_trigger_conditions = [
  {
    metric_name             = "aom_metrics_total_per_hour"
    promql                  = "label_replace(avg_over_time(aom_metrics_total_per_hour{type=\"custom\"}[59999ms]),\"__name__\",\"aom_metrics_total_per_hour\",\"\",\"\")"
    promql_for              = "3m"
    aggregate_type          = "by"
    aggregation_type        = "average"
    aggregation_window      = "1m"
    metric_statistic_method = "single"
    thresholds              = {
      "Critical" = 1
    }
    trigger_type            = "FIXED_RATE"
    trigger_interval        = "1m"
    trigger_times           = "3"
    query_param             = "{\"code\": \"a\", \"apmMetricReg\": []}"
    query_match             = "{\"id\": \"first\", \"dimension\": \"type\", \"conditionValue\": [{\"name\": \"custom\"}], \"conditionList\": [{\"name\": \"custom\"}, {\"name\": \"basic\"}], \"addMode\": \"first\", \"conditionCompare\": \"=\", \"dimensionSelected\": {\"label\": \"type\", \"id\": \"type\"}}"
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="lts_group_name=test-group" -var="alarm_rule_name=test-rule"`
2. 环境变量：`export TF_VAR_lts_group_name=test-group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建AOM防止ELB告警风暴
4. 运行 `terraform show` 查看已创建的AOM防止ELB告警风暴详情

## 参考信息

- [华为云AOM产品文档](https://support.huaweicloud.com/aom/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [AOM防止ELB告警风暴最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/aom/alarm-rule/prevent-elb-alarm-storm)
