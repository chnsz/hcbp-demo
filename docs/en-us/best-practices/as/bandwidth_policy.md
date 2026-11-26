# Deploy Bandwidth Policy

## Application Scenario

Auto Scaling (AS) bandwidth policies automatically adjust the size of a shared bandwidth when service traffic fluctuates, preventing service congestion caused by insufficient bandwidth and avoiding long-term bandwidth redundancy that wastes costs. A bandwidth policy is triggered by Cloud Eye Service (CES) alarm rules, and when the bandwidth metric meets the preset conditions, the bandwidth is automatically scaled up or down.

This best practice will introduce how to use Terraform to automatically deploy a bandwidth policy, including the creation of a shared bandwidth, a CES alarm rule, and an AS bandwidth policy, helping you manage bandwidth scaling in an Infrastructure as Code (IaC) manner.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Shared Bandwidth (huaweicloud_vpc_bandwidth)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_bandwidth)
- [CES Alarm Rule (huaweicloud_ces_alarmrule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarmrule)
- [AS Bandwidth Policy (huaweicloud_as_bandwidth_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_bandwidth_policy)

### Resource/Data Source Dependencies

```
huaweicloud_vpc_bandwidth
    ├── huaweicloud_ces_alarmrule
    └── huaweicloud_as_bandwidth_policy
            └── huaweicloud_ces_alarmrule
```

## Operation Steps

### 1. Script Preparation

Prepare a TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Shared Bandwidth

Add the following script to the TF file (such as main.tf):

```hcl
# Create a shared bandwidth resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "bandwidth_name" {
  description = "The name of the shared bandwidth"
  type        = string
}

variable "bandwidth_size" {
  description = "The size of the shared bandwidth"
  type        = number
  default     = 5
}

variable "bandwidth_type" {
  description = "The type of the shared bandwidth"
  type        = string
  default     = "share"
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = null
}

variable "charging_mode" {
  description = "The charging mode of the shared bandwidth"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the shared bandwidth"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the shared bandwidth"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "Whether to enable auto renew"
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

**Parameter Description**:
- **name**: The name of the shared bandwidth, assigned by referencing the input variable bandwidth_name
- **size**: The size of the shared bandwidth, assigned by referencing the input variable bandwidth_size, default is 5
- **bandwidth_type**: The type of the shared bandwidth, assigned by referencing the input variable bandwidth_type, default is share
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **charging_mode**: The charging mode, assigned by referencing the input variable charging_mode, default is postPaid
- **period_unit**: The period unit, assigned by referencing the input variable period_unit, valid only when charging_mode is prePaid
- **period**: The period, assigned by referencing the input variable period, valid only when charging_mode is prePaid
- **auto_renew**: Whether to enable auto renew, assigned by referencing the input variable auto_renew, default is false

### 3. Create a CES Alarm Rule

Add the following script to the TF file (such as main.tf):

```hcl
# Create a CES alarm rule resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "alarm_rule_name" {
  description = "The name of the CES alarm rule"
  type        = string
}

variable "alarm_rule_description" {
  description = "The description of the CES alarm rule"
  type        = string
  default     = ""
}

variable "alarm_enabled" {
  description = "Whether the CES alarm rule is enabled"
  type        = bool
  default     = true
}

variable "alarm_action_enabled" {
  description = "Whether to enable the action triggered by the alarm"
  type        = bool
  default     = true
}

variable "rule_conditions" {
  description = "The conditions of the alarm rule"
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

**Parameter Description**:
- **alarm_name**: The name of the alarm rule, assigned by referencing the input variable alarm_rule_name
- **alarm_description**: The description of the alarm rule, assigned by referencing the input variable alarm_rule_description
- **alarm_enabled**: Whether the alarm rule is enabled, assigned by referencing the input variable alarm_enabled
- **alarm_action_enabled**: Whether the alarm action is enabled, assigned by referencing the input variable alarm_action_enabled
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **metric.namespace**: The metric namespace, fixed to SYS.VPC
- **metric.metric_name**: The metric name, fixed to downstream_bandwidth
- **metric.dimensions.value**: The dimension value, assigned by referencing the ID of the shared bandwidth
- **condition**: The alarm conditions, assigned by referencing the input variable rule_conditions, including period, filter, comparison operator, threshold, count, unit, alarm level, and suppress duration
- **alarm_actions.type**: The alarm action type, fixed to autoscaling to trigger auto scaling

### 4. Create an AS Bandwidth Policy

Add the following script to the TF file (such as main.tf):

```hcl
# Create an AS bandwidth policy resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "bandwidth_policy_name" {
  description = "The name of the AS bandwidth policy"
  type        = string
}

variable "bandwidth_policy_type" {
  description = "The type of the AS bandwidth policy"
  type        = string
  default     = "INTERVAL_ALARM"
}

variable "bandwidth_policy_description" {
  description = "The description of the AS bandwidth policy"
  type        = string
  default     = ""
}

variable "cool_down_time" {
  description = "The cooldown period of the AS bandwidth policy"
  type        = number
  default     = 300
}

variable "interval_alarm_conditions" {
  description = "The interval alarm conditions of the bandwidth policy"
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

**Parameter Description**:
- **scaling_policy_name**: The name of the bandwidth policy, assigned by referencing the input variable bandwidth_policy_name
- **scaling_policy_type**: The type of the bandwidth policy, assigned by referencing the input variable bandwidth_policy_type, default is INTERVAL_ALARM
- **bandwidth_id**: The bandwidth ID, assigned by referencing the ID of the shared bandwidth
- **alarm_id**: The alarm rule ID, assigned by referencing the ID of the CES alarm rule
- **description**: The description of the bandwidth policy, assigned by referencing the input variable bandwidth_policy_description
- **cool_down_time**: The cooldown period, assigned by referencing the input variable cool_down_time, default is 300 seconds
- **interval_alarm_actions**: The interval alarm actions, assigned by referencing the input variable interval_alarm_conditions, including lower bound, upper bound, operation, size, and limits

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

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

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="bandwidth_name=my-bandwidth"`
2. Environment variables: `export TF_VAR_bandwidth_name=my-bandwidth`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the bandwidth policy
4. Run `terraform show` to view the created bandwidth policy

## Reference Information

- [Huawei Cloud Auto Scaling Product Documentation](https://support.huaweicloud.com/as/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For AS Bandwidth Policy](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/as/bandwidth-policy)
