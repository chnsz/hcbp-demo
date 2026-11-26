# 部署带宽策略

## 应用场景

弹性伸缩（Auto Scaling，AS）带宽策略用于在业务流量波动时自动调整共享带宽的大小，避免因带宽不足导致业务拥塞，也避免带宽长期冗余造成成本浪费。带宽策略依赖云监控服务（CES）的告警规则触发，当带宽监控指标满足预设条件时，自动执行带宽扩容或缩容动作。

本最佳实践将介绍如何使用Terraform自动化部署带宽策略，包括共享带宽、CES告警规则和AS带宽策略的创建，帮助您基于基础设施即代码（IaC）的方式实现带宽的弹性伸缩管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [共享带宽（huaweicloud_vpc_bandwidth）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_bandwidth)
- [CES告警规则（huaweicloud_ces_alarmrule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarmrule)
- [AS带宽策略（huaweicloud_as_bandwidth_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_bandwidth_policy)

### 资源/数据源依赖关系

```
huaweicloud_vpc_bandwidth
    ├── huaweicloud_ces_alarmrule
    └── huaweicloud_as_bandwidth_policy
            └── huaweicloud_ces_alarmrule
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建共享带宽

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建共享带宽资源
variable "bandwidth_name" {
  description = "共享带宽的名称"
  type        = string
}

variable "bandwidth_size" {
  description = "共享带宽的大小"
  type        = number
  default     = 5
}

variable "bandwidth_type" {
  description = "共享带宽的类型"
  type        = string
  default     = "share"
}

variable "enterprise_project_id" {
  description = "企业项目ID"
  type        = string
  default     = null
}

variable "charging_mode" {
  description = "共享带宽的计费模式"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "共享带宽的计费周期单位"
  type        = string
  default     = null
}

variable "period" {
  description = "共享带宽的计费周期"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "是否开启自动续费"
  type        = string
  default     = "false"
}

resource "huaweicloud_vpc_bandwidth" "test" {
  name                  = var.bandwidth_name
  size                  = var.bandwidth_size
  bandwidth_type        = var.bandwidth_type
  enterprise_project_id = var.enterprise_project_id
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew
}
```

**参数说明**：
- **name**：共享带宽名称，通过引用输入变量 bandwidth_name 进行赋值
- **size**：共享带宽大小，通过引用输入变量 bandwidth_size 进行赋值，默认值为5
- **bandwidth_type**：共享带宽类型，通过引用输入变量 bandwidth_type 进行赋值，默认值为share
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **charging_mode**：计费模式，通过引用输入变量 charging_mode 进行赋值，默认值为postPaid
- **period_unit**：计费周期单位，通过引用输入变量 period_unit 进行赋值，仅在计费模式为prePaid时生效
- **period**：计费周期，通过引用输入变量 period 进行赋值，仅在计费模式为prePaid时生效
- **auto_renew**：是否自动续费，通过引用输入变量 auto_renew 进行赋值，默认值为false

### 3. 创建CES告警规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CES告警规则资源
variable "alarm_rule_name" {
  description = "CES告警规则的名称"
  type        = string
}

variable "alarm_rule_description" {
  description = "CES告警规则的描述"
  type        = string
  default     = ""
}

variable "alarm_enabled" {
  description = "是否启用CES告警规则"
  type        = bool
  default     = true
}

variable "alarm_action_enabled" {
  description = "是否启用告警触发的动作"
  type        = bool
  default     = true
}

variable "rule_conditions" {
  description = "告警规则的条件"
  type        = list(object({
    period              = number
    filter              = string
    comparison_operator = string
    value               = number
    count               = number
    unit                = optional(string, null)
    alarm_level         = optional(number, 2)
    suppress_duration   = optional(number, 0)
  }))

  nullable = false

  validation {
    condition     = length(var.rule_conditions) > 0
    error_message = "The 'rule_conditions' is not allowed to be empty"
  }
}

resource "huaweicloud_ces_alarmrule" "test" {
  alarm_name            = var.alarm_rule_name
  alarm_description     = var.alarm_rule_description
  alarm_enabled         = var.alarm_enabled
  alarm_action_enabled  = var.alarm_action_enabled
  enterprise_project_id = var.enterprise_project_id

  metric {
    namespace   = "SYS.VPC"
    metric_name = "downstream_bandwidth"

    dimensions {
      name  = "bandwidth_id"
      value = huaweicloud_vpc_bandwidth.test.id
    }
  }

  dynamic "condition" {
    for_each = var.rule_conditions

    content {
      period              = condition.value.period
      filter              = condition.value.filter
      comparison_operator = condition.value.comparison_operator
      value               = condition.value.value
      count               = condition.value.count
      unit                = condition.value.unit
      alarm_level         = condition.value.alarm_level
      suppress_duration   = condition.value.suppress_duration
    }
  }

  alarm_actions {
    type              = "autoscaling"
    notification_list = []
  }
}
```

**参数说明**：
- **alarm_name**：告警规则名称，通过引用输入变量 alarm_rule_name 进行赋值
- **alarm_description**：告警规则描述，通过引用输入变量 alarm_rule_description 进行赋值
- **alarm_enabled**：是否启用告警规则，通过引用输入变量 alarm_enabled 进行赋值
- **alarm_action_enabled**：是否启用告警动作，通过引用输入变量 alarm_action_enabled 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **metric.namespace**：监控指标命名空间，固定为SYS.VPC
- **metric.metric_name**：监控指标名称，固定为downstream_bandwidth
- **metric.dimensions.value**：监控维度取值，引用共享带宽的ID进行赋值
- **condition**：告警条件，通过引用输入变量 rule_conditions 进行赋值，包括统计周期、聚合方式、比较运算符、阈值、连续次数、单位、告警级别和抑制时长
- **alarm_actions.type**：告警触发动作类型，固定为autoscaling，用于触发弹性伸缩

### 4. 创建AS带宽策略

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AS带宽策略资源
variable "bandwidth_policy_name" {
  description = "AS带宽策略的名称"
  type        = string
}

variable "bandwidth_policy_type" {
  description = "AS带宽策略的类型"
  type        = string
  default     = "INTERVAL_ALARM"
}

variable "bandwidth_policy_description" {
  description = "AS带宽策略的描述"
  type        = string
  default     = ""
}

variable "cool_down_time" {
  description = "AS带宽策略的冷却时间"
  type        = number
  default     = 300
}

variable "interval_alarm_conditions" {
  description = "带宽策略的告警区间条件"
  type        = list(object({
    lower_bound = optional(string, null)
    upper_bound = optional(string, null)
    operation   = optional(string, null)
    size        = optional(number, 2)
    limits      = optional(number, null)
  }))

  nullable = false

  validation {
    condition     = length(var.interval_alarm_conditions) > 0
    error_message = "The 'interval_alarm_conditions' is not allowed to be empty"
  }
}

resource "huaweicloud_as_bandwidth_policy" "test" {
  scaling_policy_name = var.bandwidth_policy_name
  scaling_policy_type = var.bandwidth_policy_type
  bandwidth_id        = huaweicloud_vpc_bandwidth.test.id
  alarm_id            = huaweicloud_ces_alarmrule.test.id
  description         = var.bandwidth_policy_description
  cool_down_time      = var.cool_down_time

  dynamic "interval_alarm_actions" {
    for_each = var.interval_alarm_conditions

    content {
      lower_bound = interval_alarm_actions.value.lower_bound
      upper_bound = interval_alarm_actions.value.upper_bound
      operation   = interval_alarm_actions.value.operation
      size        = interval_alarm_actions.value.size
      limits      = interval_alarm_actions.value.limits
    }
  }
}
```

**参数说明**：
- **scaling_policy_name**：带宽策略名称，通过引用输入变量 bandwidth_policy_name 进行赋值
- **scaling_policy_type**：带宽策略类型，通过引用输入变量 bandwidth_policy_type 进行赋值，默认值为INTERVAL_ALARM
- **bandwidth_id**：带宽ID，引用共享带宽的ID进行赋值
- **alarm_id**：告警规则ID，引用CES告警规则的ID进行赋值
- **description**：带宽策略描述，通过引用输入变量 bandwidth_policy_description 进行赋值
- **cool_down_time**：冷却时间，通过引用输入变量 cool_down_time 进行赋值，默认值为300秒
- **interval_alarm_actions**：告警区间动作，通过引用输入变量 interval_alarm_conditions 进行赋值，包括区间上下限、操作类型、调整大小和限制值

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
bandwidth_name  = "tf_test_bandwidth"
alarm_rule_name = "tf_test_alarm_rule"

rule_conditions = [
  {
    period              = 300
    filter              = "average"
    comparison_operator = ">"
    value               = 3
    count               = 5
    unit                = "Mbit/s"
  }
]

interval_alarm_conditions = [
  {
    lower_bound = "3"
    upper_bound = "10"
    operation   = "ADD"
    size        = 1
    limits      = 20
  },
  {
    lower_bound = "10"
    upper_bound = "20"
    operation   = "SET"
    size        = 8
  }
]

bandwidth_policy_name = "tf_test_bandwidth_policy"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="bandwidth_name=my-bandwidth"`
2. 环境变量：`export TF_VAR_bandwidth_name=my-bandwidth`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建带宽策略
4. 运行 `terraform show` 查看已创建的带宽策略

## 参考信息

- [华为云弹性伸缩产品文档](https://support.huaweicloud.com/as/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [弹性伸缩带宽策略最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/as/bandwidth-policy)
