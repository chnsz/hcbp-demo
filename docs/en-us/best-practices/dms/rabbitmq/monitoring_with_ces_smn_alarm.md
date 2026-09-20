# Deploy RabbitMQ Monitoring with CES Alarm Notifications

## Application Scenario

A Distributed Message Service (DMS) RabbitMQ instance generates key metrics such as the number of connections, channels, queues, file descriptors in use, memory usage, available disk space, and message backlog during operation. If these metrics exceed the thresholds acceptable to the business and are not detected and handled in time, message accumulation, connection exhaustion, or even instance unavailability may occur.

This best practice will introduce how to use Terraform to automatically deploy a RabbitMQ instance monitoring and alarm notification solution, including creating network resources such as VPC, subnet, and security group, deploying a RabbitMQ instance, creating an SMN topic and SMS subscription, configuring CES alarm rules to monitor key RabbitMQ metrics, and creating a CES dashboard for metric visualization. When a metric exceeds the threshold, CES sends an SMS alarm to the specified phone number through SMN, helping O&M personnel detect and handle exceptions as soon as possible.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RabbitMQ Instance Flavors (data.huaweicloud_dms_rabbitmq_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_rabbitmq_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RabbitMQ Instance (huaweicloud_dms_rabbitmq_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_instance)
- [SMN Topic (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [SMN Subscription (huaweicloud_smn_subscription)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_subscription)
- [CES Alarm Rule (huaweicloud_ces_alarmrule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarmrule)
- [CES Dashboard (huaweicloud_ces_dashboard)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_dashboard)
- [CES Dashboard Widget (huaweicloud_ces_dashboard_widget)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_dashboard_widget)

### Resource/Data Source Dependencies

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

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md) article.

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones available in the current region:

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:

- This data source requires no additional parameters, and the query result contains the names of all availability zones available in the current region.

### 3. Create a VPC

Add the following script in the TF file (such as main.tf) to create a VPC:

```hcl
# Create a VPC in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:

- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr, with a default value of 192.168.0.0/16

### 4. Create a Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create a subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:

- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC created above
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The subnet CIDR block, assigned by referencing the input variable subnet_cidr; when the variable is empty, a subnet is automatically divided from the VPC CIDR block
- **gateway_ip**: The subnet gateway IP, assigned by referencing the input variable subnet_gateway_ip; when the variable is empty, the first available IP of the subnet CIDR block is used automatically

### 5. Create a Security Group

Add the following script in the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**Parameter Description**:

- **name**: The security group name, assigned by referencing the input variable security_group_name

### 6. Create Security Group Rules

Add the following script in the TF file (such as main.tf) to create security group rules:

```hcl
# Create security group rules in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:

- **count**: The number of security group rules, dynamically created based on the length of the input variable security_group_rule_configurations
- **security_group_id**: The ID of the security group to which the rule belongs, referencing the ID of the security group created above
- **direction**: The rule direction, assigned by referencing the direction in the input variable security_group_rule_configurations
- **ethertype**: The IP protocol type, assigned by referencing the ethertype in the input variable security_group_rule_configurations
- **protocol**: The protocol type, assigned by referencing the protocol in the input variable security_group_rule_configurations
- **port_range_min**: The minimum port range, assigned by referencing the port_range_min in the input variable security_group_rule_configurations
- **port_range_max**: The maximum port range, assigned by referencing the port_range_max in the input variable security_group_rule_configurations
- **remote_ip_prefix**: The remote IP address range, assigned by referencing the remote_ip_prefix in the input variable security_group_rule_configurations
- **description**: The rule description, assigned by referencing the description in the input variable security_group_rule_configurations

### 7. Query RabbitMQ Instance Flavors

Add the following script in the TF file (such as main.tf) to query RabbitMQ instance flavors:

```hcl
# Query RabbitMQ instance flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:

- **type**: The instance flavor type, assigned by referencing the input variable instance_flavor_type, with a default value of cluster
- **storage_spec_code**: The storage specification code, assigned by referencing the input variable instance_storage_spec_code
- **availability_zones**: The availability zone list, sliced from the availability zone data source according to the input variable availability_zone_number

### 8. Create a RabbitMQ Instance

Add the following script in the TF file (such as main.tf) to create a RabbitMQ instance:

```hcl
# Create a RabbitMQ instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:

- **name**: The RabbitMQ instance name, assigned by referencing the input variable instance_name
- **engine_version**: The instance engine version, assigned by referencing the input variable instance_engine_version
- **flavor_id**: The instance flavor ID, referencing the first flavor ID queried from the RabbitMQ instance flavors data source
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC created above
- **network_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet created above
- **security_group_id**: The ID of the security group to which the instance belongs, referencing the ID of the security group created above
- **availability_zones**: The availability zone list to which the instance belongs, sliced from the availability zone data source according to the input variable availability_zone_number
- **broker_num**: The number of brokers of the instance, assigned by referencing the input variable instance_broker_num
- **storage_space**: The storage space of the instance (in GB), assigned by referencing the input variable instance_storage_space
- **storage_spec_code**: The storage specification code of the instance, assigned by referencing the input variable instance_storage_spec_code
- **ssl_enable**: Whether to enable SSL, assigned by referencing the input variable instance_ssl_enable
- **access_user**: The access user of the instance, assigned by referencing the input variable instance_access_user_name
- **password**: The access password of the instance, assigned by referencing the input variable instance_password
- **description**: The instance description, assigned by referencing the input variable instance_description
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **tags**: The instance tags, assigned by referencing the input variable instance_tags
- **charging_mode**: The charging mode, assigned by referencing the input variable charging_mode
- **period_unit**: The period unit, assigned by referencing the input variable period_unit
- **period**: The period, assigned by referencing the input variable period
- **auto_renew**: Whether to enable auto renew, assigned by referencing the input variable auto_renew

### 9. Create an SMN Topic

Add the following script in the TF file (such as main.tf) to create an SMN topic:

```hcl
# Create an SMN topic in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:

- **name**: The SMN topic name, assigned by referencing the input variable smn_topic_name
- **display_name**: The topic display name, assigned by referencing the input variable smn_topic_display_name; when the variable is empty, the topic name is used as the display name
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id

### 10. Create an SMN SMS Subscription

Add the following script in the TF file (such as main.tf) to create an SMN SMS subscription:

```hcl
# Create an SMN SMS subscription in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:

- **topic_urn**: The URN of the SMN topic to which the subscription belongs, referencing the ID of the SMN topic created above
- **protocol**: The subscription protocol, fixed to sms, indicating SMS notification
- **endpoint**: The subscription endpoint, that is, the phone number for receiving SMS, assigned by referencing the input variable sms_subscription_endpoint, in the format +[country code][phone number]
- **remark**: The subscription remark, assigned by referencing the input variable sms_subscription_remark

### 11. Create a CES Alarm Rule

Add the following script in the TF file (such as main.tf) to create a CES alarm rule:

```hcl
# Create a CES alarm rule in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:

- **alarm_name**: The alarm rule name, assigned by referencing the input variable alarm_rule_name
- **alarm_description**: The alarm rule description, assigned by referencing the input variable alarm_rule_description
- **alarm_action_enabled**: Whether to enable the action triggered by an alarm, assigned by referencing the input variable alarm_action_enabled
- **alarm_enabled**: Whether to enable the alarm rule, assigned by referencing the input variable alarm_enabled
- **alarm_type**: The alarm type, fixed to MULTI_INSTANCE, indicating multi-metric multi-instance alarm
- **metric.namespace**: The metric namespace, fixed to SYS.DMS, indicating DMS service metrics
- **resources.dimensions**: The alarm resource dimension, where name is fixed to rabbitmq_instance_id and value references the ID of the RabbitMQ instance created above
- **condition**: The alarm conditions, dynamically generated by the dynamic block according to the input variable alarm_rule_conditions; each condition contains the metric name, period, filter, comparison operator, threshold, count, unit, suppress duration, and alarm level
- **alarm_actions**: The action executed when the alarm is triggered, whose notification list references the URN of the SMN topic created above
- **ok_actions**: The action executed when the alarm is recovered, whose notification list references the URN of the SMN topic created above
- **notification_begin_time**: The alarm notification start time, assigned by referencing the input variable alarm_rule_notification_begin_time
- **notification_end_time**: The alarm notification stop time, assigned by referencing the input variable alarm_rule_notification_end_time

### 12. Create a CES Dashboard

Add the following script in the TF file (such as main.tf) to create a CES dashboard:

```hcl
# Create a CES dashboard in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:

- **name**: The dashboard name, assigned by the local variable dashboard_name; when the input variable dashboard_name is empty, RabbitMQ-{instance name}-dashboard is used as the default name
- **row_widget_num**: The number of widgets displayed per row, fixed to 2
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id

### 13. Create CES Dashboard Widgets

Add the following script in the TF file (such as main.tf) to create CES dashboard widgets:

```hcl
# Create CES dashboard widgets in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:

- **count**: The number of widgets, dynamically created based on the length of the input variable dashboard_widget_configurations
- **dashboard_id**: The ID of the dashboard to which the widget belongs, referencing the ID of the dashboard created above
- **title**: The widget title, assigned by referencing the title in the input variable dashboard_widget_configurations
- **view**: The view type, fixed to line, indicating a line chart
- **metric_display_mode**: The metric display mode, fixed to single, indicating single-metric display
- **metrics.namespace**: The metric namespace, fixed to SYS.DMS
- **metrics.metric_name**: The metric name, assigned by referencing the metric_name in the input variable dashboard_widget_configurations
- **metrics.dimensions**: The metric dimension, where name is fixed to rabbitmq_instance_id and values references the ID of the RabbitMQ instance created above
- **location**: The widget location, assigned by referencing the left, top, width, and height in the input variable dashboard_widget_configurations

### 14. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Network resources
vpc_name            = "rabbitmq-monitor-vpc"
subnet_name         = "rabbitmq-monitor-subnet"
security_group_name = "rabbitmq-monitor-sg"

# RabbitMQ instance
instance_name             = "rabbitmq-monitored"
instance_access_user_name = "rabbitmq_admin"
instance_password         = "YourPassword@123"

# SMN notification
smn_topic_name            = "rabbitmq-alarm-topic"
sms_subscription_endpoint = "+8613600000000"

# CES alarm rule
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

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 15. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the RabbitMQ instance and its monitoring and alarm resources
4. Run `terraform show` to view the created RabbitMQ instance and its monitoring and alarm resources

> Note: RabbitMQ instance creation takes about 20 to 50 minutes, depending on the instance flavor and the number of brokers.

## Reference Information

- [Huawei Cloud Distributed Message Service RabbitMQ Product Documentation](https://support.huaweicloud.com/rabbitmq/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DMS RabbitMQ Monitoring with CES Alarm Notifications](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/rabbitmq/monitoring-with-ces-smn-alarm)
