# 部署RabbitMQ监控与CES告警通知

## 应用场景

分布式消息服务（DMS）RabbitMQ实例在运行过程中会产生连接数、通道数、队列数、文件描述符使用量、内存使用量、磁盘可用空间和消息堆积量等关键指标。当这些指标超过业务可接受的阈值时，若不能及时发现和处理，可能导致消息积压、连接耗尽甚至实例不可用。

本最佳实践将介绍如何使用Terraform自动化部署一套RabbitMQ实例监控与告警通知方案，包括创建VPC、子网和安全组等网络资源，部署RabbitMQ实例，创建SMN主题与短信订阅，配置CES告警规则对RabbitMQ关键指标进行监控，并创建CES监控看板用于指标的可视化展示。当指标超过阈值时，CES会通过SMN向指定手机号发送短信告警，帮助运维人员第一时间感知并处理异常。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RabbitMQ实例规格（data.huaweicloud_dms_rabbitmq_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_rabbitmq_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RabbitMQ实例（huaweicloud_dms_rabbitmq_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_instance)
- [SMN主题（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [SMN订阅（huaweicloud_smn_subscription）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_subscription)
- [CES告警规则（huaweicloud_ces_alarmrule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarmrule)
- [CES监控看板（huaweicloud_ces_dashboard）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_dashboard)
- [CES监控看板小部件（huaweicloud_ces_dashboard_widget）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_dashboard_widget)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_dms_rabbitmq_flavors
        └── huaweicloud_dms_rabbitmq_instance
            ├── huaweicloud_ces_alarmrule
            │   └── huaweicloud_ces_dashboard
            │       └── huaweicloud_ces_dashboard_widget
            └── huaweicloud_ces_dashboard_widget

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dms_rabbitmq_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_dms_rabbitmq_instance

huaweicloud_smn_topic
    ├── huaweicloud_smn_subscription
    └── huaweicloud_ces_alarmrule
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询当前region下可用的可用区：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：

- 该数据源无需额外参数，查询结果中包含当前region下所有可用的可用区名称。

### 3. 创建VPC

在TF文件（如main.tf）中添加以下脚本以创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：

- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC网段，通过引用输入变量 vpc_cidr 进行赋值，默认值为 192.168.0.0/16

### 4. 创建子网

在TF文件（如main.tf）中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建子网
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：

- **vpc_id**：子网所属的VPC ID，引用前面创建的VPC的ID
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网网段，通过引用输入变量 subnet_cidr 进行赋值；当该变量为空时，自动从VPC网段中划分一个子网
- **gateway_ip**：子网网关IP，通过引用输入变量 subnet_gateway_ip 进行赋值；当该变量为空时，自动使用子网网段的第一个可用IP

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**参数说明**：

- **name**：安全组名称，通过引用输入变量 security_group_name 进行赋值

### 6. 创建安全组规则

在TF文件（如main.tf）中添加以下脚本以创建安全组规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组规则
variable "security_group_rule_configurations" {
  description = "The list of security group rule configurations."

  type = list(object({
    direction        = string
    ethertype        = string
    protocol         = string
    port_range_min   = number
    port_range_max   = number
    remote_ip_prefix = string
    description      = string
  }))

  default = [
    {
      direction        = "ingress"
      ethertype        = "IPv4"
      protocol         = "tcp"
      port_range_min   = 5672
      port_range_max   = 5672
      remote_ip_prefix = "192.168.0.0/16"
      description      = "Allow access to RabbitMQ AMQP port"
    },
    {
      direction        = "ingress"
      ethertype        = "IPv4"
      protocol         = "tcp"
      port_range_min   = 15672
      port_range_max   = 15672
      remote_ip_prefix = "192.168.0.0/16"
      description      = "Allow access to RabbitMQ management port"
    }
  ]
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = length(var.security_group_rule_configurations)

  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = lookup(var.security_group_rule_configurations[count.index], "direction", null)
  ethertype         = lookup(var.security_group_rule_configurations[count.index], "ethertype", null)
  protocol          = lookup(var.security_group_rule_configurations[count.index], "protocol", null)
  port_range_min    = lookup(var.security_group_rule_configurations[count.index], "port_range_min", null)
  port_range_max    = lookup(var.security_group_rule_configurations[count.index], "port_range_max", null)
  remote_ip_prefix  = lookup(var.security_group_rule_configurations[count.index], "remote_ip_prefix", null)
  description       = lookup(var.security_group_rule_configurations[count.index], "description", null)
}
```

**参数说明**：

- **count**：安全组规则数量，根据输入变量 security_group_rule_configurations 的长度动态创建
- **security_group_id**：安全组规则所属的安全组ID，引用前面创建的安全组的ID
- **direction**：规则方向，通过引用输入变量 security_group_rule_configurations 中的 direction 进行赋值
- **ethertype**：IP协议类型，通过引用输入变量 security_group_rule_configurations 中的 ethertype 进行赋值
- **protocol**：协议类型，通过引用输入变量 security_group_rule_configurations 中的 protocol 进行赋值
- **port_range_min**：端口范围最小值，通过引用输入变量 security_group_rule_configurations 中的 port_range_min 进行赋值
- **port_range_max**：端口范围最大值，通过引用输入变量 security_group_rule_configurations 中的 port_range_max 进行赋值
- **remote_ip_prefix**：远端IP地址范围，通过引用输入变量 security_group_rule_configurations 中的 remote_ip_prefix 进行赋值
- **description**：规则描述，通过引用输入变量 security_group_rule_configurations 中的 description 进行赋值

### 7. 查询RabbitMQ实例规格

在TF文件（如main.tf）中添加以下脚本以查询RabbitMQ实例规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询RabbitMQ实例规格
variable "instance_flavor_type" {
  description = "The flavor type of the RabbitMQ instance"
  type        = string
  default     = "cluster"
}

variable "instance_storage_spec_code" {
  description = "The storage specification code of the RabbitMQ instance"
  type        = string
  default     = "dms.physical.storage.ultra.v2"
}

variable "availability_zone_number" {
  description = "The number of availability zones to which the RabbitMQ instance belongs"
  type        = number
  default     = 1
}

data "huaweicloud_dms_rabbitmq_flavors" "test" {
  type               = var.instance_flavor_type
  storage_spec_code  = var.instance_storage_spec_code
  availability_zones = try(slice(data.huaweicloud_availability_zones.test.names, 0, var.availability_zone_number), null)
}
```

**参数说明**：

- **type**：实例规格类型，通过引用输入变量 instance_flavor_type 进行赋值，默认值为 cluster
- **storage_spec_code**：存储规格编码，通过引用输入变量 instance_storage_spec_code 进行赋值
- **availability_zones**：可用区列表，根据输入变量 availability_zone_number 从可用区数据源中截取相应数量的可用区

### 8. 创建RabbitMQ实例

在TF文件（如main.tf）中添加以下脚本以创建RabbitMQ实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RabbitMQ实例
variable "instance_name" {
  description = "The name of the RabbitMQ instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the RabbitMQ instance"
  type        = string
  default     = "3.8.35"
}

variable "instance_broker_num" {
  description = "The number of brokers of the RabbitMQ instance"
  type        = number
  default     = 3
}

variable "instance_storage_space" {
  description = "The storage space of the RabbitMQ instance (in GB)"
  type        = number
  default     = 600
}

variable "instance_ssl_enable" {
  description = "Whether to enable SSL for the RabbitMQ instance"
  type        = bool
  default     = false
}

variable "instance_access_user_name" {
  description = "The access user of the RabbitMQ instance"
  type        = string
}

variable "instance_password" {
  description = "The access password of the RabbitMQ instance"
  type        = string
  sensitive   = true
}

variable "instance_description" {
  description = "The description of the RabbitMQ instance"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = null
}

variable "instance_tags" {
  description = "The key/value pairs to associate with the RabbitMQ instance"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the RabbitMQ instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the RabbitMQ instance"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the RabbitMQ instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the RabbitMQ instance"
  type        = string
  default     = "false"
}

resource "huaweicloud_dms_rabbitmq_instance" "test" {
  name                  = var.instance_name
  engine_version        = var.instance_engine_version
  flavor_id             = try(data.huaweicloud_dms_rabbitmq_flavors.test.flavors[0].id, null)
  vpc_id                = huaweicloud_vpc.test.id
  network_id            = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zones    = try(slice(data.huaweicloud_availability_zones.test.names, 0, var.availability_zone_number), null)
  broker_num            = var.instance_broker_num
  storage_space         = var.instance_storage_space
  storage_spec_code     = var.instance_storage_spec_code
  ssl_enable            = var.instance_ssl_enable
  access_user           = var.instance_access_user_name
  password              = var.instance_password
  description           = var.instance_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.instance_tags
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew

  lifecycle {
    ignore_changes = [
      flavor_id,
      availability_zones,
    ]
  }
}
```

**参数说明**：

- **name**：RabbitMQ实例名称，通过引用输入变量 instance_name 进行赋值
- **engine_version**：实例引擎版本，通过引用输入变量 instance_engine_version 进行赋值
- **flavor_id**：实例规格ID，引用RabbitMQ实例规格数据源中查询到的第一个规格ID
- **vpc_id**：实例所属的VPC ID，引用前面创建的VPC的ID
- **network_id**：实例所属的子网ID，引用前面创建的子网的ID
- **security_group_id**：实例所属的安全组ID，引用前面创建的安全组的ID
- **availability_zones**：实例所属的可用区列表，根据输入变量 availability_zone_number 从可用区数据源中截取
- **broker_num**：实例的broker数量，通过引用输入变量 instance_broker_num 进行赋值
- **storage_space**：实例的存储空间（GB），通过引用输入变量 instance_storage_space 进行赋值
- **storage_spec_code**：实例的存储规格编码，通过引用输入变量 instance_storage_spec_code 进行赋值
- **ssl_enable**：是否启用SSL，通过引用输入变量 instance_ssl_enable 进行赋值
- **access_user**：实例的访问用户名，通过引用输入变量 instance_access_user_name 进行赋值
- **password**：实例的访问密码，通过引用输入变量 instance_password 进行赋值
- **description**：实例描述，通过引用输入变量 instance_description 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **tags**：实例标签，通过引用输入变量 instance_tags 进行赋值
- **charging_mode**：计费模式，通过引用输入变量 charging_mode 进行赋值
- **period_unit**：包周期单位，通过引用输入变量 period_unit 进行赋值
- **period**：包周期时长，通过引用输入变量 period 进行赋值
- **auto_renew**：是否自动续费，通过引用输入变量 auto_renew 进行赋值

### 9. 创建SMN主题

在TF文件（如main.tf）中添加以下脚本以创建SMN主题：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN主题
variable "smn_topic_name" {
  description = "The name of the SMN topic used to send alarm notifications"
  type        = string
}

variable "smn_topic_display_name" {
  description = "The display name of the SMN topic"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_smn_topic" "test" {
  name                  = var.smn_topic_name
  display_name          = var.smn_topic_display_name != "" ? var.smn_topic_display_name : var.smn_topic_name
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：

- **name**：SMN主题名称，通过引用输入变量 smn_topic_name 进行赋值
- **display_name**：主题显示名称，通过引用输入变量 smn_topic_display_name 进行赋值；当该变量为空时，使用主题名称作为显示名称
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值

### 10. 创建SMN短信订阅

在TF文件（如main.tf）中添加以下脚本以创建SMN短信订阅：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN短信订阅
variable "sms_subscription_endpoint" {
  description = "The phone number for SMS notification, format: +[country code][phone number], e.g. +8613600000000"
  type        = string
}

variable "sms_subscription_remark" {
  description = "The remark of the SMS subscription"
  type        = string
  default     = "RabbitMQ alarm notification"
}

resource "huaweicloud_smn_subscription" "test" {
  topic_urn = huaweicloud_smn_topic.test.id
  protocol  = "sms"
  endpoint  = var.sms_subscription_endpoint
  remark    = var.sms_subscription_remark
}
```

**参数说明**：

- **topic_urn**：订阅所属的SMN主题URN，引用前面创建的SMN主题的ID
- **protocol**：订阅协议，固定为 sms，表示短信通知
- **endpoint**：订阅终端，即接收短信的手机号，通过引用输入变量 sms_subscription_endpoint 进行赋值，格式为 +[国家码][手机号]
- **remark**：订阅备注，通过引用输入变量 sms_subscription_remark 进行赋值

### 11. 创建CES告警规则

在TF文件（如main.tf）中添加以下脚本以创建CES告警规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CES告警规则
variable "alarm_rule_name" {
  description = "The name of the CES alarm rule"
  type        = string
}

variable "alarm_rule_description" {
  description = "The description of the CES alarm rule"
  type        = string
  default     = "The alarm rule for RabbitMQ monitoring"
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

variable "alarm_rule_conditions" {
  description = "The list of alarm rule conditions for RabbitMQ monitoring"

  type = list(object({
    metric_name         = string
    period              = number
    filter              = string
    comparison_operator = string
    value               = number
    count               = number
    unit                = optional(string)
    suppress_duration   = optional(number, 300)
    alarm_level         = optional(number, 2)
  }))

  nullable = false
}

variable "alarm_rule_notification_begin_time" {
  description = "The alarm notification start time, e.g. 00:00"
  type        = string
  default     = null
}

variable "alarm_rule_notification_end_time" {
  description = "The alarm notification stop time, e.g. 23:59"
  type        = string
  default     = null
}

resource "huaweicloud_ces_alarmrule" "test" {
  alarm_name            = var.alarm_rule_name
  alarm_description     = var.alarm_rule_description
  alarm_action_enabled  = var.alarm_action_enabled
  alarm_enabled         = var.alarm_enabled
  alarm_type            = "MULTI_INSTANCE"
  enterprise_project_id = var.enterprise_project_id

  metric {
    namespace = "SYS.DMS"
  }

  resources {
    dimensions {
      name  = "rabbitmq_instance_id"
      value = huaweicloud_dms_rabbitmq_instance.test.id
    }
  }

  dynamic "condition" {
    for_each = var.alarm_rule_conditions

    content {
      metric_name         = condition.value.metric_name
      period              = condition.value.period
      filter              = condition.value.filter
      comparison_operator = condition.value.comparison_operator
      value               = condition.value.value
      count               = condition.value.count
      unit                = condition.value.unit
      suppress_duration   = condition.value.suppress_duration
      alarm_level         = condition.value.alarm_level
    }
  }

  alarm_actions {
    type = "notification"

    notification_list = [
      huaweicloud_smn_topic.test.topic_urn
    ]
  }

  ok_actions {
    type = "notification"

    notification_list = [
      huaweicloud_smn_topic.test.topic_urn
    ]
  }

  notification_begin_time = var.alarm_rule_notification_begin_time
  notification_end_time   = var.alarm_rule_notification_end_time

  depends_on = [huaweicloud_dms_rabbitmq_instance.test]
}
```

**参数说明**：

- **alarm_name**：告警规则名称，通过引用输入变量 alarm_rule_name 进行赋值
- **alarm_description**：告警规则描述，通过引用输入变量 alarm_rule_description 进行赋值
- **alarm_action_enabled**：是否启用告警触发动作，通过引用输入变量 alarm_action_enabled 进行赋值
- **alarm_enabled**：是否启用告警规则，通过引用输入变量 alarm_enabled 进行赋值
- **alarm_type**：告警类型，固定为 MULTI_INSTANCE，表示多指标多实例告警
- **metric.namespace**：指标命名空间，固定为 SYS.DMS，表示DMS服务指标
- **resources.dimensions**：告警资源维度，name 固定为 rabbitmq_instance_id，value 引用前面创建的RabbitMQ实例的ID
- **condition**：告警条件，通过动态块根据输入变量 alarm_rule_conditions 动态生成，每个条件包含指标名称、统计周期、统计方式、比较运算符、阈值、连续触发次数、单位、抑制时长和告警级别
- **alarm_actions**：告警触发时执行的动作，通知列表引用前面创建的SMN主题URN
- **ok_actions**：告警恢复时执行的动作，通知列表引用前面创建的SMN主题URN
- **notification_begin_time**：告警通知开始时间，通过引用输入变量 alarm_rule_notification_begin_time 进行赋值
- **notification_end_time**：告警通知结束时间，通过引用输入变量 alarm_rule_notification_end_time 进行赋值

### 12. 创建CES监控看板

在TF文件（如main.tf）中添加以下脚本以创建CES监控看板：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CES监控看板
variable "dashboard_name" {
  description = "The name of the CES monitoring dashboard for RabbitMQ"
  type        = string
  default     = ""
  nullable    = false
}

locals {
  dashboard_name = var.dashboard_name != "" ? var.dashboard_name : "RabbitMQ-${var.instance_name}-dashboard"
}

resource "huaweicloud_ces_dashboard" "test" {
  name                  = local.dashboard_name
  row_widget_num        = 2
  enterprise_project_id = var.enterprise_project_id

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_ces_alarmrule.test
  ]
}
```

**参数说明**：

- **name**：监控看板名称，通过本地变量 dashboard_name 进行赋值；当输入变量 dashboard_name 为空时，使用 RabbitMQ-{实例名称}-dashboard 作为默认名称
- **row_widget_num**：每行展示的小部件数量，固定为 2
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值

### 13. 创建CES监控看板小部件

在TF文件（如main.tf）中添加以下脚本以创建CES监控看板小部件：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CES监控看板小部件
variable "dashboard_widget_configurations" {
  description = "The list of dashboard widget configurations for RabbitMQ monitoring"

  type = list(object({
    title       = string
    metric_name = string
    left        = number
    top         = number
    width       = optional(number, 6)
    height      = optional(number, 3)
  }))

  default = [
    {
      title       = "RabbitMQ Connections"
      metric_name = "connections"
      left        = 0
      top         = 0
    },
    {
      title       = "RabbitMQ Channels"
      metric_name = "channels"
      left        = 6
      top         = 0
    },
    {
      title       = "RabbitMQ Queues"
      metric_name = "queues"
      left        = 0
      top         = 3
    },
    {
      title       = "RabbitMQ File Descriptors Used"
      metric_name = "fd_used"
      left        = 6
      top         = 3
    },
  ]
}

resource "huaweicloud_ces_dashboard_widget" "test" {
  count = length(var.dashboard_widget_configurations)

  dashboard_id        = huaweicloud_ces_dashboard.test.id
  title               = var.dashboard_widget_configurations[count.index].title
  view                = "line"
  metric_display_mode = "single"

  metrics {
    namespace   = "SYS.DMS"
    metric_name = var.dashboard_widget_configurations[count.index].metric_name

    dimensions {
      name        = "rabbitmq_instance_id"
      filter_type = "specific_instances"
      values      = [huaweicloud_dms_rabbitmq_instance.test.id]
    }
  }

  location {
    left   = var.dashboard_widget_configurations[count.index].left
    top    = var.dashboard_widget_configurations[count.index].top
    width  = var.dashboard_widget_configurations[count.index].width
    height = var.dashboard_widget_configurations[count.index].height
  }
}
```

**参数说明**：

- **count**：小部件数量，根据输入变量 dashboard_widget_configurations 的长度动态创建
- **dashboard_id**：小部件所属的监控看板ID，引用前面创建的监控看板的ID
- **title**：小部件标题，通过引用输入变量 dashboard_widget_configurations 中的 title 进行赋值
- **view**：视图类型，固定为 line，表示折线图
- **metric_display_mode**：指标展示模式，固定为 single，表示单指标展示
- **metrics.namespace**：指标命名空间，固定为 SYS.DMS
- **metrics.metric_name**：指标名称，通过引用输入变量 dashboard_widget_configurations 中的 metric_name 进行赋值
- **metrics.dimensions**：指标维度，name 固定为 rabbitmq_instance_id，values 引用前面创建的RabbitMQ实例的ID
- **location**：小部件位置，通过引用输入变量 dashboard_widget_configurations 中的 left、top、width 和 height 进行赋值

### 14. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 网络资源
vpc_name            = "rabbitmq-monitor-vpc"
subnet_name         = "rabbitmq-monitor-subnet"
security_group_name = "rabbitmq-monitor-sg"

# RabbitMQ实例
instance_name             = "rabbitmq-monitored"
instance_access_user_name = "rabbitmq_admin"
instance_password         = "YourPassword@123"

# SMN通知
smn_topic_name            = "rabbitmq-alarm-topic"
sms_subscription_endpoint = "+8613600000000"

# CES告警规则
alarm_rule_name = "rabbitmq-instance-alarm"

alarm_rule_conditions = [
  {
    metric_name         = "connections"
    period              = 300
    filter              = "average"
    comparison_operator = ">"
    value               = 1000
    count               = 3
    unit                = "count"
    suppress_duration   = 3600
    alarm_level         = 2
  },
  {
    metric_name         = "channels"
    period              = 300
    filter              = "average"
    comparison_operator = ">"
    value               = 5000
    count               = 3
    unit                = "count"
    suppress_duration   = 3600
    alarm_level         = 2
  },
  {
    metric_name         = "queues"
    period              = 300
    filter              = "average"
    comparison_operator = "="
    value               = 0
    count               = 3
    unit                = "count"
    suppress_duration   = 3600
    alarm_level         = 3
  },
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 15. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建RabbitMQ实例及其监控告警资源
4. 运行 `terraform show` 查看已创建的RabbitMQ实例及其监控告警资源

> 注意：RabbitMQ实例创建大约需要20至50分钟，具体时间取决于实例规格和broker数量。

## 参考信息

- [华为云分布式消息服务RabbitMQ产品文档](https://support.huaweicloud.com/rabbitmq/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RabbitMQ监控与CES告警通知最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/rabbitmq/monitoring-with-ces-smn-alarm)
