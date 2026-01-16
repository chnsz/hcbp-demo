# Deploy CES Event Alarm Rule

## Application Scenario

Simple Message Notification (SMN) is a reliable and scalable message notification service provided by Huawei Cloud, supporting multiple message notification methods including email, SMS, HTTP/HTTPS, etc. Cloud Eye Service (CES) is a monitoring service provided by Huawei Cloud, supporting real-time resource monitoring and alarms. Through CES event alarm rules, SMN topic events can be monitored, and notifications are automatically sent when alarm conditions are met. This best practice introduces how to use Terraform to automatically deploy CES event alarm rules, including creating SMN topics and configuring CES alarm rules, achieving monitoring and alerting for SMN topic events.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [SMN Topic Resource (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [CES Alarm Rule Resource (huaweicloud_ces_alarmrule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarmrule)

### Resource/Data Source Dependencies

```text
huaweicloud_smn_topic
    └── huaweicloud_ces_alarmrule
```

> Note: CES alarm rules need to reference the SMN topic URN to send alarm notifications, so alarm rules depend on SMN topic resources.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create SMN Topic

Add the following script to the TF file (such as main.tf) to create an SMN topic:

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

# Create SMN topic resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_smn_topic" "test" {
  name                  = var.smn_topic_name
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **name**: Topic name, assigned by referencing the input variable `smn_topic_name`
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable `enterprise_project_id`, optional parameter

### 3. Create CES Alarm Rule

Add the following script to the TF file (such as main.tf) to create a CES alarm rule:

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

# Create CES alarm rule resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
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

**Parameter Description**:
- **alarm_name**: Alarm rule name, assigned by referencing the input variable `alarm_rule_name`
- **alarm_description**: Alarm rule description, assigned by referencing the input variable `alarm_rule_description`, optional parameter
- **alarm_action_enabled**: Whether to enable alarm actions, assigned by referencing the input variable `alarm_action_enabled`, default is `true`
- **alarm_enabled**: Whether to enable the alarm, assigned by referencing the input variable `alarm_enabled`, default is `true`
- **alarm_type**: Alarm type, assigned by referencing the input variable `alarm_type`, default is `ALL_INSTANCE`
- **metric**: Monitoring metric configuration, namespace set to `SYS.SMN` indicates monitoring SMN service
- **condition**: Alarm condition list, creates multiple alarm conditions through dynamic block `dynamic "condition"` based on input variable `alarm_rule_conditions`
- **resources**: Monitoring resource dimension list, creates resource dimensions through dynamic block `dynamic "resources"` based on input variable `alarm_rule_resource`
- **alarm_actions**: Alarm action configuration, type is `notification`, notification list references SMN topic URN
- **notification_begin_time**: Alarm notification start time, assigned by referencing the input variable `alarm_rule_notification_begin_time`, optional parameter
- **notification_end_time**: Alarm notification stop time, assigned by referencing the input variable `alarm_rule_notification_end_time`, optional parameter
- **effective_timezone**: Time zone, assigned by referencing the input variable `alarm_rule_effective_timezone`, optional parameter

> Note: Alarm rules reference the SMN topic `topic_urn` through `notification_list` in `alarm_actions` to send alarm notifications. Alarm conditions support multiple condition configurations, each condition includes metric name, period, filter method, comparison operator, threshold, and other parameters.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# SMN topic configuration (Required)
smn_topic_name = "tf_test_topic"

# CES alarm rule basic information (Required)
alarm_rule_name        = "tf_test_alarm_rule"
alarm_rule_description = "Monitor SMN topic events"
alarm_type             = "ALL_INSTANCE"

# Alarm condition configuration (Required)
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

# Monitoring resource dimension configuration (Optional)
alarm_rule_resource = [
  {
    name = "topic_id"
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="smn_topic_name=tf_test_topic" -var="alarm_rule_name=tf_test_alarm_rule"`
2. Environment variables: `export TF_VAR_smn_topic_name=tf_test_topic`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating SMN topic and CES alarm rule
4. Run `terraform show` to view the created resources

## Reference Information

- [Huawei Cloud SMN Product Documentation](https://support.huaweicloud.com/smn/index.html)
- [Huawei Cloud CES Product Documentation](https://support.huaweicloud.com/ces/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CES Event Alarm Rule](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/smn/ces-event-alarm-rule)
