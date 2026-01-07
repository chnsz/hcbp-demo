# 部署AOM按标签分发告警

## 应用场景

应用运维管理（Application Operations Management，AOM）是华为云提供的一站式应用运维管理平台，支持应用监控、日志管理、告警管理等核心功能。通过配置AOM按标签分发告警，可以根据华为云标签（Tag）对告警进行分组和分发，实现基于标签的精细化告警管理。这种方式可以帮助用户根据业务需求对不同类型的资源进行差异化告警处理，提高告警管理的灵活性和效率。

本最佳实践将介绍如何使用Terraform自动化部署AOM按标签分发告警，包括创建Prometheus实例、云服务接入、告警规则，以及配置告警动作规则和标签管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [DCS实例查询数据源（data.huaweicloud_dcs_instances）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_instances)
- [项目查询数据源（data.huaweicloud_identity_projects）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### 资源

- [云日志服务日志组资源（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [云日志服务日志流资源（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [消息通知服务主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [消息通知服务日志转储资源（huaweicloud_smn_logtank）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_logtank)
- [AOM告警动作规则资源（huaweicloud_aom_alarm_action_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_alarm_action_rule)
- [标签管理服务资源标签资源（huaweicloud_tms_resource_tags）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/tms_resource_tags)
- [AOM Prometheus实例资源（huaweicloud_aom_prom_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_prom_instance)
- [AOM云服务接入资源（huaweicloud_aom_cloud_service_access）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_cloud_service_access)
- [AOM告警规则资源（huaweicloud_aomv4_alarm_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aomv4_alarm_rule)

### 资源/数据源依赖关系

```
data.huaweicloud_dcs_instances
    ├── huaweicloud_lts_group
    │   ├── huaweicloud_lts_stream
    │   │   └── huaweicloud_smn_logtank
    │   └── huaweicloud_smn_logtank
    ├── huaweicloud_tms_resource_tags
    │   └── huaweicloud_aom_prom_instance
    │       └── huaweicloud_aom_cloud_service_access
    │           └── huaweicloud_aomv4_alarm_rule
    └── huaweicloud_smn_topic
        ├── huaweicloud_smn_logtank
        └── huaweicloud_aom_alarm_action_rule
            └── huaweicloud_aomv4_alarm_rule

data.huaweicloud_identity_projects
    └── huaweicloud_tms_resource_tags
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询DCS实例信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于获取DCS实例的企业项目ID等信息：

```hcl
variable "dcs_instance_name" {
  description = "The name of the existing DCS instance to be monitored"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下指定名称的DCS实例信息，用于获取企业项目ID等信息
data "huaweicloud_dcs_instances" "test" {
  name = var.dcs_instance_name
}
```

**参数说明**：
- **name**：DCS实例名称，通过引用输入变量dcs_instance_name进行赋值

### 3. 通过数据源查询项目信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于获取项目ID：

```hcl
variable "region_name" {
  description = "The region where resources will be created"
  type        = string
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下指定名称的项目信息，用于获取项目ID
data "huaweicloud_identity_projects" "test" {
  name = var.region_name
}
```

**参数说明**：
- **name**：项目名称，通过引用输入变量region_name进行赋值

### 4. 创建云日志服务日志组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云日志服务日志组资源：

```hcl
variable "lts_group_name" {
  description = "The name of the LTS group used to store SMN notification logs"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云日志服务日志组资源
resource "huaweicloud_lts_group" "test" {
  group_name            = var.lts_group_name
  ttl_in_days           = 30
  enterprise_project_id = local.enterprise_project_id
}
```

**参数说明**：
- **group_name**：日志组名称，通过引用输入变量lts_group_name进行赋值
- **ttl_in_days**：日志保存时间（单位：天），设置为30天
- **enterprise_project_id**：企业项目ID，通过local变量enterprise_project_id进行赋值

### 5. 创建云日志服务日志流资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云日志服务日志流资源：

```hcl
variable "lts_stream_name" {
  description = "The name of the LTS stream used to store SMN notification logs"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云日志服务日志流资源
resource "huaweicloud_lts_stream" "test" {
  group_id              = huaweicloud_lts_group.test.id
  stream_name           = var.lts_stream_name
  enterprise_project_id = local.enterprise_project_id
}
```

**参数说明**：
- **group_id**：日志组ID，引用前面创建的云日志服务日志组资源（huaweicloud_lts_group.test）的ID
- **stream_name**：日志流名称，通过引用输入变量lts_stream_name进行赋值
- **enterprise_project_id**：企业项目ID，通过local变量enterprise_project_id进行赋值

### 6. 创建消息通知服务主题资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建消息通知服务主题资源：

```hcl
variable "smn_topic_name" {
  description = "The name of the SMN topic used to send notifications"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建消息通知服务主题资源
resource "huaweicloud_smn_topic" "test" {
  name                  = var.smn_topic_name
  enterprise_project_id = local.enterprise_project_id
}
```

**参数说明**：
- **name**：主题名称，通过引用输入变量smn_topic_name进行赋值
- **enterprise_project_id**：企业项目ID，通过local变量enterprise_project_id进行赋值

### 7. 创建消息通知服务日志转储资源

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

### 8. 创建AOM告警动作规则资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建AOM告警动作规则资源：

```hcl
variable "alarm_action_rule_name" {
  description = "The name of the AOM alarm action rule used to send SMN notifications"
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
- **name**：告警动作规则名称，通过引用输入变量alarm_action_rule_name进行赋值
- **user_name**：用户名，通过引用输入变量alarm_action_rule_user_name进行赋值
- **type**：告警动作规则类型，通过引用输入变量alarm_action_rule_type进行赋值，默认值为"1"（表示通知类型）
- **notification_template**：通知模板名称，使用内置模板"aom.built-in.template.zh"
- **smn_topics.topic_urn**：SMN主题URN，引用前面创建的消息通知服务主题资源（huaweicloud_smn_topic.test）的topic_urn

### 9. 创建标签管理服务资源标签资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建标签管理服务资源标签资源：

```hcl
variable "alarm_rule_matric_dimension_tags" {
  description = "The custom tags to be added to the DCS instance for alarm distribution"
  type        = map(string)
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建标签管理服务资源标签资源
resource "huaweicloud_tms_resource_tags" "test" {
  project_id = local.exact_project_id

  resources {
    resource_type = "dcs"
    resource_id   = try(data.huaweicloud_dcs_instances.test.instances[0].id, null)
  }

  tags = var.alarm_rule_matric_dimension_tags
}
```

**参数说明**：
- **project_id**：项目ID，通过local变量exact_project_id进行赋值
- **resources.resource_type**：资源类型，设置为"dcs"
- **resources.resource_id**：资源ID，根据DCS实例查询数据源（data.huaweicloud_dcs_instances.test）的返回结果进行赋值
- **tags**：标签键值对，通过引用输入变量alarm_rule_matric_dimension_tags进行赋值

### 10. 创建AOM Prometheus实例资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建AOM Prometheus实例资源：

```hcl
variable "prometheus_instance_name" {
  description = "The name of the Prometheus instance for cloud services"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AOM Prometheus实例资源
resource "huaweicloud_aom_prom_instance" "test" {
  depends_on = [huaweicloud_tms_resource_tags.test]

  prom_name             = var.prometheus_instance_name
  prom_type             = "CLOUD_SERVICE"
  enterprise_project_id = local.enterprise_project_id
}
```

**参数说明**：
- **depends_on**：显式依赖关系，确保标签管理服务资源标签资源先于Prometheus实例资源创建
- **prom_name**：Prometheus实例名称，通过引用输入变量prometheus_instance_name进行赋值
- **prom_type**：Prometheus实例类型，设置为"CLOUD_SERVICE"（表示云服务类型）
- **enterprise_project_id**：企业项目ID，通过local变量enterprise_project_id进行赋值

### 11. 创建AOM云服务接入资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建AOM云服务接入资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AOM云服务接入资源
resource "huaweicloud_aom_cloud_service_access" "test" {
  instance_id           = huaweicloud_aom_prom_instance.test.id
  service               = "DCS"
  tag_sync              = "auto"
  enterprise_project_id = local.enterprise_project_id

  provisioner "local-exec" {
    command = "sleep 240" # 等待接入中心完成连接并生成指标
  }
}
```

**参数说明**：
- **instance_id**：Prometheus实例ID，引用前面创建的AOM Prometheus实例资源（huaweicloud_aom_prom_instance.test）的ID
- **service**：云服务类型，设置为"DCS"
- **tag_sync**：标签同步方式，设置为"auto"（表示自动同步）
- **enterprise_project_id**：企业项目ID，通过local变量enterprise_project_id进行赋值
- **provisioner.local-exec.command**：本地执行命令，等待240秒以确保接入中心完成连接并生成指标

### 12. 创建AOM告警规则资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建AOM告警规则资源：

```hcl
variable "alarm_rule_name" {
  description = "The name of the AOM alarm rule"
  type        = string
}

variable "alarm_rule_trigger_conditions" {
  description = "The trigger conditions of the AOM alarm rule"
  type = list(object({
    metric_name             = string
    promql                  = string
    promql_for              = optional(string, "")
    aggregate_type          = optional(string, "by")
    aggregation_type        = string
    aggregation_window      = string
    metric_unit             = string
    metric_query_mode       = string
    metric_namespace        = string
    operator                = string
    metric_statistic_method = string
    thresholds              = map(string)
    trigger_type            = string
    trigger_interval        = string
    trigger_times           = string
    query_param             = string
    query_match             = string
  }))
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AOM告警规则资源
resource "huaweicloud_aomv4_alarm_rule" "test" {
  depends_on = [huaweicloud_aom_cloud_service_access.test]

  name                  = var.alarm_rule_name
  type                  = "metric"
  enable                = true
  prom_instance_id      = huaweicloud_aom_prom_instance.test.id
  enterprise_project_id = local.enterprise_project_id

  alarm_notifications {
    notification_enable       = true
    notification_type         = "direct"
    bind_notification_rule_id = huaweicloud_aom_alarm_action_rule.test.id
    notify_resolved           = true
    notify_triggered          = true
    notify_frequency          = "0"
  }

  metric_alarm_spec {
    monitor_type = "all_metric"

    recovery_conditions {
      recovery_timeframe = 1
    }

    dynamic "trigger_conditions" {
      for_each = var.alarm_rule_trigger_conditions

      content {
        metric_query_mode       = trigger_conditions.value.metric_query_mode
        metric_name             = trigger_conditions.value.metric_name
        promql                  = trigger_conditions.value.promql
        promql_for              = trigger_conditions.value.promql_for
        aggregate_type          = trigger_conditions.value.aggregate_type
        aggregation_type        = trigger_conditions.value.aggregation_type
        aggregation_window      = trigger_conditions.value.aggregation_window
        metric_unit             = trigger_conditions.value.metric_unit
        metric_namespace        = trigger_conditions.value.metric_namespace
        operator                = trigger_conditions.value.operator
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
- **depends_on**：显式依赖关系，确保AOM云服务接入资源先于告警规则资源创建
- **name**：告警规则名称，通过引用输入变量alarm_rule_name进行赋值
- **type**：告警规则类型，设置为"metric"（表示指标类型）
- **enable**：是否启用告警规则，设置为true
- **prom_instance_id**：Prometheus实例ID，引用前面创建的AOM Prometheus实例资源（huaweicloud_aom_prom_instance.test）的ID
- **enterprise_project_id**：企业项目ID，通过local变量enterprise_project_id进行赋值
- **alarm_notifications.notification_enable**：是否启用通知，设置为true
- **alarm_notifications.notification_type**：通知类型，设置为"direct"（表示直接通知）
- **alarm_notifications.bind_notification_rule_id**：绑定的通知规则ID，引用前面创建的AOM告警动作规则资源（huaweicloud_aom_alarm_action_rule.test）的ID
- **alarm_notifications.notify_resolved**：是否通知恢复，设置为true
- **alarm_notifications.notify_triggered**：是否通知触发，设置为true
- **alarm_notifications.notify_frequency**：通知频率，设置为"0"（表示每次触发都通知）
- **metric_alarm_spec.monitor_type**：监控类型，设置为"all_metric"（表示所有指标）
- **metric_alarm_spec.recovery_conditions.recovery_timeframe**：恢复时间范围，设置为1（单位：分钟）
- **metric_alarm_spec.trigger_conditions**：触发条件列表，通过dynamic块根据输入变量alarm_rule_trigger_conditions动态生成，其中promql必须包含标签条件（如`huaweicloud_sys_dcs_cpu_usage{Ihn="OPEN"}`），query_match必须包含标签匹配条件

### 13. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# DCS实例配置
dcs_instance_name = "tf_test_aom_alarm_rule_distribute_alarm"

# 云日志服务配置
lts_group_name  = "tf_test_aom_alarm_rule_distribute_alarm"
lts_stream_name = "tf_test_aom_alarm_rule_distribute_alarm"

# 消息通知服务主题配置
smn_topic_name = "tf_test_aom_alarm_rule_distribute_alarm"

# AOM告警动作规则配置
alarm_action_rule_name      = "tf_test_aom_alarm_rule_distribute_alarm_by_Ihn_tag"
alarm_action_rule_user_name = "servicestage"

# 标签配置
alarm_rule_matric_dimension_tags = {
  "Ihn" = "OPEN"
}

# Prometheus实例配置
prometheus_instance_name = "tf_test_aom_alarm_rule_distribute_alarm"

# AOM告警规则配置
alarm_rule_name = "tf_test_aom_alarm_rule_distribute_alarm_by_Ihn_tag"
alarm_rule_trigger_conditions = [
  {
    metric_name             = "huaweicloud_sys_dcs_cpu_usage"
    promql                  = "label_replace(avg_over_time(huaweicloud_sys_dcs_cpu_usage{Ihn=\"OPEN\"}[59999ms]),\"__name__\",\"huaweicloud_sys_dcs_cpu_usage\",\"\",\"\")"
    promql_for              = ""
    aggregate_type          = "by"
    aggregation_type        = "average"
    aggregation_window      = "1m"
    metric_unit             = "%"
    metric_query_mode       = "PROM"
    metric_namespace        = "SYS.DCS"
    operator                = ">"
    metric_statistic_method = "single"
    thresholds              = {
      "Critical" = 1
    }
    trigger_type            = "FIXED_RATE"
    trigger_interval        = "1m"
    trigger_times           = "3"
    query_param             = "{\"code\": \"a\", \"apmMetricReg\": []}"
    query_match             = "[{\"id\":\"first\",\"dimension\":\"Ihn\",\"conditionValue\":[{\"name\":\"OPEN\"}],\"conditionList\":[{\"name\":\"OPEN\"}],\"addMode\": \"first\",\"conditionCompare\":\"=\",\"regExpress\":null,\"dimensionSelected\":{\"label\":\"Ihn\",\"id\":\"Ihn\"}}]"
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="dcs_instance_name=test-instance" -var="lts_group_name=test-group"`
2. 环境变量：`export TF_VAR_dcs_instance_name=test-instance`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 14. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建AOM按标签分发告警
4. 运行 `terraform show` 查看已创建的AOM按标签分发告警详情

## 参考信息

- [华为云AOM产品文档](https://support.huaweicloud.com/aom/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [AOM按标签分发告警最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/aom/alarm-rule/distribute-alarm)
