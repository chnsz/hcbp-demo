# Deploy AOM Distribute Alarms by Tags

## Application Scenario

Application Operations Management (AOM) is a one-stop application operations management platform provided by Huawei Cloud, supporting core functions such as application monitoring, log management, and alarm management. By configuring AOM to distribute alarms by tags, alarms can be grouped and distributed based on Huawei Cloud tags (Tag), enabling fine-grained alarm management based on tags. This approach helps users perform differentiated alarm processing for different types of resources according to business needs, improving the flexibility and efficiency of alarm management.

This best practice will introduce how to use Terraform to automatically deploy AOM distribute alarms by tags, including creating Prometheus instances, cloud service access, alarm rules, and configuring alarm action rules and tag management.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [DCS Instance Query Data Source (data.huaweicloud_dcs_instances)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_instances)
- [Project Query Data Source (data.huaweicloud_identity_projects)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### Resources

- [Log Tank Service Log Group Resource (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [Log Tank Service Log Stream Resource (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [Simple Message Notification Topic Resource (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [Simple Message Notification Log Tank Resource (huaweicloud_smn_logtank)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_logtank)
- [AOM Alarm Action Rule Resource (huaweicloud_aom_alarm_action_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_alarm_action_rule)
- [Tag Management Service Resource Tags Resource (huaweicloud_tms_resource_tags)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/tms_resource_tags)
- [AOM Prometheus Instance Resource (huaweicloud_aom_prom_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_prom_instance)
- [AOM Cloud Service Access Resource (huaweicloud_aom_cloud_service_access)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_cloud_service_access)
- [AOM Alarm Rule Resource (huaweicloud_aomv4_alarm_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aomv4_alarm_rule)

### Resource/Data Source Dependencies

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

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query DCS Instance Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the query results are used to obtain the enterprise project ID and other information of the DCS instance:

```hcl
variable "dcs_instance_name" {
  description = "The name of the existing DCS instance to be monitored"
  type        = string
  default     = ""
}

# Query DCS instance information with the specified name in the specified region (defaults to the region specified in the provider block when region parameter is omitted), used to obtain enterprise project ID and other information
data "huaweicloud_dcs_instances" "test" {
  name = var.dcs_instance_name
}
```

**Parameter Description**:
- **name**: The DCS instance name, assigned by referencing the input variable dcs_instance_name

### 3. Query Project Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the query results are used to obtain the project ID:

```hcl
variable "region_name" {
  description = "The region where resources will be created"
  type        = string
}

# Query project information with the specified name in the specified region (defaults to the region specified in the provider block when region parameter is omitted), used to obtain project ID
data "huaweicloud_identity_projects" "test" {
  name = var.region_name
}
```

**Parameter Description**:
- **name**: The project name, assigned by referencing the input variable region_name

### 4. Create Log Tank Service Log Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Log Tank Service log group resource:

```hcl
variable "lts_group_name" {
  description = "The name of the LTS group used to store SMN notification logs"
  type        = string
}

# Create Log Tank Service log group resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_lts_group" "test" {
  group_name            = var.lts_group_name
  ttl_in_days           = 30
  enterprise_project_id = local.enterprise_project_id
}
```

**Parameter Description**:
- **group_name**: The log group name, assigned by referencing the input variable lts_group_name
- **ttl_in_days**: The log retention time (unit: days), set to 30 days
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the local variable enterprise_project_id

### 5. Create Log Tank Service Log Stream Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Log Tank Service log stream resource:

```hcl
variable "lts_stream_name" {
  description = "The name of the LTS stream used to store SMN notification logs"
  type        = string
}

# Create Log Tank Service log stream resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_lts_stream" "test" {
  group_id              = huaweicloud_lts_group.test.id
  stream_name           = var.lts_stream_name
  enterprise_project_id = local.enterprise_project_id
}
```

**Parameter Description**:
- **group_id**: The log group ID, referencing the ID of the previously created Log Tank Service log group resource (huaweicloud_lts_group.test)
- **stream_name**: The log stream name, assigned by referencing the input variable lts_stream_name
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the local variable enterprise_project_id

### 6. Create Simple Message Notification Topic Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Simple Message Notification topic resource:

```hcl
variable "smn_topic_name" {
  description = "The name of the SMN topic used to send notifications"
  type        = string
}

# Create Simple Message Notification topic resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_smn_topic" "test" {
  name                  = var.smn_topic_name
  enterprise_project_id = local.enterprise_project_id
}
```

**Parameter Description**:
- **name**: The topic name, assigned by referencing the input variable smn_topic_name
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the local variable enterprise_project_id

### 7. Create Simple Message Notification Log Tank Resource

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

### 8. Create AOM Alarm Action Rule Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an AOM alarm action rule resource:

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
- **name**: The alarm action rule name, assigned by referencing the input variable alarm_action_rule_name
- **user_name**: The user name, assigned by referencing the input variable alarm_action_rule_user_name
- **type**: The alarm action rule type, assigned by referencing the input variable alarm_action_rule_type, default value is "1" (indicating notification type)
- **notification_template**: The notification template name, using the built-in template "aom.built-in.template.zh"
- **smn_topics.topic_urn**: The SMN topic URN, referencing the topic_urn of the previously created Simple Message Notification topic resource (huaweicloud_smn_topic.test)

### 9. Create Tag Management Service Resource Tags Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Tag Management Service resource tags resource:

```hcl
variable "alarm_rule_matric_dimension_tags" {
  description = "The custom tags to be added to the DCS instance for alarm distribution"
  type        = map(string)
}

# Create Tag Management Service resource tags resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_tms_resource_tags" "test" {
  project_id = local.exact_project_id

  resources {
    resource_type = "dcs"
    resource_id   = try(data.huaweicloud_dcs_instances.test.instances[0].id, null)
  }

  tags = var.alarm_rule_matric_dimension_tags
}
```

**Parameter Description**:
- **project_id**: The project ID, assigned by referencing the local variable exact_project_id
- **resources.resource_type**: The resource type, set to "dcs"
- **resources.resource_id**: The resource ID, assigned based on the return results of the DCS instance query data source (data.huaweicloud_dcs_instances.test)
- **tags**: The tag key-value pairs, assigned by referencing the input variable alarm_rule_matric_dimension_tags

### 10. Create AOM Prometheus Instance Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an AOM Prometheus instance resource:

```hcl
variable "prometheus_instance_name" {
  description = "The name of the Prometheus instance for cloud services"
  type        = string
}

# Create AOM Prometheus instance resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_aom_prom_instance" "test" {
  depends_on = [huaweicloud_tms_resource_tags.test]

  prom_name             = var.prometheus_instance_name
  prom_type             = "CLOUD_SERVICE"
  enterprise_project_id = local.enterprise_project_id
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency relationship, ensuring the Tag Management Service resource tags resource is created before the Prometheus instance resource
- **prom_name**: The Prometheus instance name, assigned by referencing the input variable prometheus_instance_name
- **prom_type**: The Prometheus instance type, set to "CLOUD_SERVICE" (indicating cloud service type)
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the local variable enterprise_project_id

### 11. Create AOM Cloud Service Access Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an AOM cloud service access resource:

```hcl
# Create AOM cloud service access resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_aom_cloud_service_access" "test" {
  instance_id           = huaweicloud_aom_prom_instance.test.id
  service               = "DCS"
  tag_sync              = "auto"
  enterprise_project_id = local.enterprise_project_id

  provisioner "local-exec" {
    command = "sleep 240" # Waiting for the access center to complete the connection and generate indicators
  }
}
```

**Parameter Description**:
- **instance_id**: The Prometheus instance ID, referencing the ID of the previously created AOM Prometheus instance resource (huaweicloud_aom_prom_instance.test)
- **service**: The cloud service type, set to "DCS"
- **tag_sync**: The tag synchronization method, set to "auto" (indicating automatic synchronization)
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the local variable enterprise_project_id
- **provisioner.local-exec.command**: Local execution command, wait 240 seconds to ensure the access center completes the connection and generates indicators

### 12. Create AOM Alarm Rule Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an AOM alarm rule resource:

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

# Create AOM alarm rule resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
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
      metric_alarm_spec # If you want to update this configuration, please use a version higher than 1.82.3
    ]
  }
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency relationship, ensuring the AOM cloud service access resource is created before the alarm rule resource
- **name**: The alarm rule name, assigned by referencing the input variable alarm_rule_name
- **type**: The alarm rule type, set to "metric" (indicating metric type)
- **enable**: Whether to enable the alarm rule, set to true
- **prom_instance_id**: The Prometheus instance ID, referencing the ID of the previously created AOM Prometheus instance resource (huaweicloud_aom_prom_instance.test)
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the local variable enterprise_project_id
- **alarm_notifications.notification_enable**: Whether to enable notifications, set to true
- **alarm_notifications.notification_type**: The notification type, set to "direct" (indicating direct notification)
- **alarm_notifications.bind_notification_rule_id**: The bound notification rule ID, referencing the ID of the previously created AOM alarm action rule resource (huaweicloud_aom_alarm_action_rule.test)
- **alarm_notifications.notify_resolved**: Whether to notify on recovery, set to true
- **alarm_notifications.notify_triggered**: Whether to notify on trigger, set to true
- **alarm_notifications.notify_frequency**: The notification frequency, set to "0" (indicating notify on every trigger)
- **metric_alarm_spec.monitor_type**: The monitoring type, set to "all_metric" (indicating all metrics)
- **metric_alarm_spec.recovery_conditions.recovery_timeframe**: The recovery time frame, set to 1 (unit: minutes)
- **metric_alarm_spec.trigger_conditions**: The trigger conditions list, dynamically generated through the dynamic block based on the input variable alarm_rule_trigger_conditions, where promql must include tag conditions (e.g., `huaweicloud_sys_dcs_cpu_usage{Ihn="OPEN"}`), and query_match must include tag matching conditions

### 13. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# DCS Instance Configuration
dcs_instance_name = "tf_test_aom_alarm_rule_distribute_alarm"

# Log Tank Service Configuration
lts_group_name  = "tf_test_aom_alarm_rule_distribute_alarm"
lts_stream_name = "tf_test_aom_alarm_rule_distribute_alarm"

# Simple Message Notification Topic Configuration
smn_topic_name = "tf_test_aom_alarm_rule_distribute_alarm"

# AOM Alarm Action Rule Configuration
alarm_action_rule_name      = "tf_test_aom_alarm_rule_distribute_alarm_by_Ihn_tag"
alarm_action_rule_user_name = "servicestage"

# Tag Configuration
alarm_rule_matric_dimension_tags = {
  "Ihn" = "OPEN"
}

# Prometheus Instance Configuration
prometheus_instance_name = "tf_test_aom_alarm_rule_distribute_alarm"

# AOM Alarm Rule Configuration
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

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="dcs_instance_name=test-instance" -var="lts_group_name=test-group"`
2. Environment variables: `export TF_VAR_dcs_instance_name=test-instance`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 14. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating AOM distribute alarms by tags
4. Run `terraform show` to view the details of the created AOM distribute alarms by tags

## Reference Information

- [Huawei Cloud AOM Product Documentation](https://support.huaweicloud.com/aom/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For AOM Distribute Alarms by Tags](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/aom/alarm-rule/distribute-alarm)
