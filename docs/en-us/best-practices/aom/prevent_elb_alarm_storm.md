# Deploy AOM Prevent ELB Alarm Storm

## Application Scenario

Application Operations Management (AOM) is a one-stop application operations management platform provided by Huawei Cloud, supporting core functions such as application monitoring, log management, and alarm management. When monitoring ELB business layer metrics, a large number of duplicate or similar alarms may be generated, causing alarm storms that affect operational efficiency. By configuring AOM alarm group rules, similar alarms can be grouped and merged, reducing alarm noise and preventing alarm storms, improving the effectiveness of alarm management.

This best practice will introduce how to use Terraform to automatically deploy AOM prevent ELB alarm storm, including creating LTS log groups and streams, SMN topics and log tanks, AOM alarm action rules, alarm group rules, and configuring alarm rules.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Log Tank Service Log Group Resource (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [Log Tank Service Log Stream Resource (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [Simple Message Notification Topic Resource (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [Simple Message Notification Log Tank Resource (huaweicloud_smn_logtank)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_logtank)
- [AOM Alarm Action Rule Resource (huaweicloud_aom_alarm_action_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_alarm_action_rule)
- [AOM Alarm Group Rule Resource (huaweicloud_aom_alarm_group_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_alarm_group_rule)
- [AOM Alarm Rule Resource (huaweicloud_aomv4_alarm_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aomv4_alarm_rule)

### Resource/Data Source Dependencies

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

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Log Tank Service Log Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Log Tank Service log group resource:

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

# Create Log Tank Service log group resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_lts_group" "test" {
  group_name            = var.lts_group_name
  ttl_in_days           = 30
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:
- **group_name**: The log group name, assigned by referencing the input variable lts_group_name
- **ttl_in_days**: The log retention time (unit: days), set to 30 days
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, set to null when the value is an empty string

### 3. Create Log Tank Service Log Stream Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Log Tank Service log stream resource:

```hcl
variable "lts_stream_name" {
  description = "The name of the LTS stream that used to store the SMN notification logs"
  type        = string
}

# Create Log Tank Service log stream resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_lts_stream" "test" {
  group_id              = huaweicloud_lts_group.test.id
  stream_name           = var.lts_stream_name
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:
- **group_id**: The log group ID, referencing the ID of the previously created Log Tank Service log group resource (huaweicloud_lts_group.test)
- **stream_name**: The log stream name, assigned by referencing the input variable lts_stream_name
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, set to null when the value is an empty string

### 4. Create Simple Message Notification Topic Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Simple Message Notification topic resource:

```hcl
variable "smn_topic_name" {
  description = "The name of the SMN topic that used to send the SMN notification"
  type        = string
}

# Create Simple Message Notification topic resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_smn_topic" "test" {
  name                  = var.smn_topic_name
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:
- **name**: The topic name, assigned by referencing the input variable smn_topic_name
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, set to null when the value is an empty string

### 5. Create Simple Message Notification Log Tank Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Simple Message Notification log tank resource:

```hcl
# Create Simple Message Notification log tank resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_smn_logtank" "test" {
  topic_urn     = huaweicloud_smn_topic.test.topic_urn
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id
}
```

**Parameter Description**:
- **topic_urn**: The topic URN, referencing the topic_urn of the previously created Simple Message Notification topic resource (huaweicloud_smn_topic.test)
- **log_group_id**: The log group ID, referencing the ID of the previously created Log Tank Service log group resource (huaweicloud_lts_group.test)
- **log_stream_id**: The log stream ID, referencing the ID of the previously created Log Tank Service log stream resource (huaweicloud_lts_stream.test)

### 6. Create AOM Alarm Action Rule Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an AOM alarm action rule resource:

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

# Create AOM alarm action rule resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
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

**Parameter Description**:
- **name**: The alarm action rule name, assigned by referencing the input variable alarm_action_rule_name, default value is "apm"
- **user_name**: The user name, assigned by referencing the input variable alarm_action_rule_user_name
- **type**: The alarm action rule type, assigned by referencing the input variable alarm_action_rule_type, default value is "1" (indicating notification type)
- **notification_template**: The notification template name, using the built-in template "aom.built-in.template.zh"
- **smn_topics.topic_urn**: The SMN topic URN, referencing the topic_urn of the previously created Simple Message Notification topic resource (huaweicloud_smn_topic.test)

### 7. Create AOM Alarm Group Rule Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an AOM alarm group rule resource:

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

# Create AOM alarm group rule resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
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

**Parameter Description**:
- **depends_on**: Explicit dependency relationship, ensuring the AOM alarm action rule resource is created before the alarm group rule resource
- **name**: The alarm group rule name, assigned by referencing the input variable alarm_group_rule_name
- **group_by**: The list of grouping fields, set to ["resource_provider"] (indicating grouping by resource provider)
- **group_interval**: The group check interval (unit: seconds), assigned by referencing the input variable alarm_group_rule_group_interval, default value is 60 seconds
- **group_repeat_waiting**: The group repeat waiting time (unit: seconds), assigned by referencing the input variable alarm_group_rule_group_repeat_waiting, default value is 3600 seconds
- **group_wait**: The group wait time (unit: seconds), assigned by referencing the input variable alarm_group_rule_group_wait, default value is 15 seconds
- **description**: The alarm group rule description, assigned by referencing the input variable alarm_group_rule_description, set to null when the value is an empty string
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, set to null when the value is an empty string
- **detail.bind_notification_rule_ids**: The list of bound notification rule IDs, referencing the name of the previously created AOM alarm action rule resource (huaweicloud_aom_alarm_action_rule.test)
- **detail.match**: The list of matching conditions, dynamically generated through the dynamic block based on the input variable alarm_group_rule_condition_matching_rules, default filters for Critical and Major severity alarms and alarms from AOM

### 8. Create AOM Alarm Rule Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an AOM alarm rule resource:

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

# Create AOM alarm rule resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
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
      metric_alarm_spec # If you want to update this configuration, please use a version higher than 1.82.3
    ]
  }
}
```

**Parameter Description**:
- **name**: The alarm rule name, assigned by referencing the input variable alarm_rule_name
- **type**: The alarm rule type, set to "metric" (indicating metric type)
- **enable**: Whether to enable the alarm rule, set to true
- **prom_instance_id**: The Prometheus instance ID, assigned by referencing the input variable prometheus_instance_id, default value is "0" (indicating the default Prometheus_AOM_Default instance)
- **alarm_notifications.notification_enable**: Whether to enable notifications, set to true
- **alarm_notifications.notification_type**: The notification type, set to "alarm_policy" (indicating alarm policy type)
- **alarm_notifications.route_group_enable**: Whether to enable route grouping, set to true
- **alarm_notifications.route_group_rule**: The route group rule name, referencing the name of the previously created AOM alarm group rule resource (huaweicloud_aom_alarm_group_rule.test)
- **alarm_notifications.notify_resolved**: Whether to notify on recovery, set to true
- **alarm_notifications.notify_triggered**: Whether to notify on trigger, set to true
- **alarm_notifications.notify_frequency**: The notification frequency, set to "-1" (indicating using the alarm group rule's frequency settings)
- **metric_alarm_spec.monitor_type**: The monitoring type, set to "all_metric" (indicating all metrics)
- **metric_alarm_spec.recovery_conditions.recovery_timeframe**: The recovery time frame, set to 1 (unit: minutes)
- **metric_alarm_spec.trigger_conditions**: The trigger conditions list, dynamically generated through the dynamic block based on the input variable alarm_rule_trigger_conditions

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Log Tank Service Configuration
lts_group_name  = "tf_test_aom_prevent_elb_alarm_storm"
lts_stream_name = "tf_test_aom_prevent_elb_alarm_storm"

# Simple Message Notification Topic Configuration
smn_topic_name = "tf_test_aom_prevent_elb_alarm_storm"

# AOM Alarm Action Rule Configuration
alarm_action_rule_user_name = "servicestage"

# AOM Alarm Group Rule Configuration
alarm_group_rule_name = "tf_test_aom_prevent_elb_alarm_storm"

# AOM Alarm Rule Configuration
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

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="lts_group_name=test-group" -var="alarm_rule_name=test-rule"`
2. Environment variables: `export TF_VAR_lts_group_name=test-group`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating AOM prevent ELB alarm storm
4. Run `terraform show` to view the details of the created AOM prevent ELB alarm storm

## Reference Information

- [Huawei Cloud AOM Product Documentation](https://support.huaweicloud.com/aom/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For AOM Prevent ELB Alarm Storm](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/aom/alarm-rule/prevent-elb-alarm-storm)
