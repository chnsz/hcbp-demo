# Deploy SQL Alarm Rule

## Application Scenario

Huawei Cloud Log Tank Service (LTS) SQL alarm rule functionality allows users to set alarm conditions based on SQL query results, automatically triggering alarm notifications when query results meet preset conditions. By configuring SQL alarm rules, you can implement real-time log data monitoring, anomaly detection, and automated alerting, improving operation efficiency and system reliability.

This best practice is particularly suitable for scenarios that require real-time log data monitoring, system anomaly detection, and automated alert notification, such as application performance monitoring, error log alerts, business metric monitoring, security event detection, etc. This best practice will introduce how to use Terraform to automatically deploy LTS SQL alarm rules, including SMN topic, log group, log stream, and SQL alarm rule creation, implementing a complete log monitoring and alert solution.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [LTS Notification Template Query Data Source (data.huaweicloud_lts_notification_templates)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/lts_notification_templates)

### Resources

- [SMN Topic Resource (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [Log Group Resource (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [Log Stream Resource (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [SQL Alarm Rule Resource (huaweicloud_lts_sql_alarm_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_sql_alarm_rule)

### Resource/Data Source Dependencies

```
data.huaweicloud_lts_notification_templates
    └── huaweicloud_lts_sql_alarm_rule

huaweicloud_smn_topic
    └── huaweicloud_lts_sql_alarm_rule

huaweicloud_lts_group
    ├── huaweicloud_lts_stream
    └── huaweicloud_lts_sql_alarm_rule

huaweicloud_lts_stream
    └── huaweicloud_lts_sql_alarm_rule
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create SMN Topic

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an SMN topic resource:

```hcl
variable "topic_name" {
  description = "The name of the SMN topic"
  type        = string
}

# Create an SMN topic resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_smn_topic" "test" {
  name         = var.topic_name
  display_name = "The display name of topic"
}
```

**Parameter Description**:
- **name**: SMN topic name, assigned by referencing the input variable topic_name
- **display_name**: SMN topic display name, set to "The display name of topic"

### 3. Create Log Group

Add the following script to the TF file to instruct Terraform to create a log group resource:

```hcl
variable "group_name" {
  description = "The name of the log group"
  type        = string
}

variable "group_log_expiration_days" {
  description = "The log expiration days of the log group"
  type        = number
  default     = 14
}

# Create a log group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_lts_group" "test" {
  group_name  = var.group_name
  ttl_in_days = var.group_log_expiration_days
}
```

**Parameter Description**:
- **group_name**: Log group name, assigned by referencing the input variable group_name
- **ttl_in_days**: Log expiration days for the log group, assigned by referencing the input variable group_log_expiration_days, default value is 14

### 4. Create Log Stream

Add the following script to the TF file to instruct Terraform to create a log stream resource:

```hcl
variable "stream_name" {
  description = "The name of the log stream"
  type        = string
}

variable "stream_log_expiration_days" {
  description = "The log expiration days of the log stream"
  type        = number
  default     = null
}

# Create a log stream resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.stream_name
  ttl_in_days = var.stream_log_expiration_days
}
```

**Parameter Description**:
- **group_id**: Log group ID that the log stream belongs to, referencing the ID of the previously created log group resource
- **stream_name**: Log stream name, assigned by referencing the input variable stream_name
- **ttl_in_days**: Log expiration days for the log stream, assigned by referencing the input variable stream_log_expiration_days, default value is null (inherits log group settings)

### 5. Query LTS Notification Template Information

Add the following script to the TF file to instruct Terraform to query LTS notification template information:

```hcl
variable "notification_template_name" {
  description = "The name of the notification template"
  type        = string
  default     = ""
  nullable    = false
}

variable "domain_id" {
  description = "The domain ID"
  type        = string
  default     = null
}

# Get all LTS notification template information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) that meets specific conditions, used to create SQL alarm rules
data "huaweicloud_lts_notification_templates" "test" {
  count = var.notification_template_name != "" ? 0 : 1

  domain_id = var.domain_id
}
```

**Parameter Description**:
- **count**: Conditional creation, creates this data source when notification_template_name variable is not an empty string
- **domain_id**: Domain ID, assigned by referencing the input variable domain_id, default value is null

### 6. Create SQL Alarm Rule

Add the following script to the TF file to instruct Terraform to create a SQL alarm rule resource:

```hcl
variable "alarm_rule_name" {
  description = "The name of the SQL alarm rule"
  type        = string
}

variable "alarm_rule_condition_expression" {
  description = "The condition expression of the SQL alarm rule"
  type        = string
}

variable "alarm_rule_alarm_level" {
  description = "The alarm level of the SQL alarm rule"
  type        = string
  default     = "MINOR"
}

variable "alarm_rule_trigger_condition_count" {
  description = "The trigger condition count of the SQL alarm rule"
  type        = number
  default     = 2
}

variable "alarm_rule_trigger_condition_frequency" {
  description = "The trigger condition frequency of the SQL alarm rule"
  type        = number
  default     = 3
}

variable "alarm_rule_send_recovery_notifications" {
  description = "The send recovery notifications of the SQL alarm rule"
  type        = bool
  default     = true
}

variable "alarm_rule_recovery_frequency" {
  description = "The recovery frequency of the SQL alarm rule"
  type        = number
  default     = 4
}

variable "alarm_rule_notification_frequency" {
  description = "The notification frequency of the SQL alarm rule"
  type        = number
  default     = 15
}

variable "alarm_rule_alias" {
  description = "The alias of the SQL alarm rule"
  type        = string
  default     = ""
}

variable "alarm_rule_request_title" {
  description = "The request title of the SQL alarm rule"
  type        = string
}

variable "alarm_rule_request_sql" {
  description = "The request SQL of the SQL alarm rule"
  type        = string
}

variable "alarm_rule_request_search_time_range_unit" {
  description = "The request search time range unit of the SQL alarm rule"
  type        = string
  default     = "minute"
}

variable "alarm_rule_request_search_time_range" {
  description = "The request search time range of the SQL alarm rule"
  type        = number
  default     = 5
}

variable "alarm_rule_frequency_type" {
  description = "The frequency type of the SQL alarm rule"
  type        = string
  default     = "HOURLY"
}

variable "alarm_rule_notification_user_name" {
  description = "The notification user name of the SQL alarm rule"
  type        = string
}

variable "alarm_rule_notification_language" {
  description = "The notification language of the SQL alarm rule"
  type        = string
  default     = "en-us"
}

# Create a SQL alarm rule resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_lts_sql_alarm_rule" "test" {
  name                        = var.alarm_rule_name
  condition_expression        = var.alarm_rule_condition_expression
  alarm_level                 = var.alarm_rule_alarm_level
  send_notifications          = true
  trigger_condition_count     = var.alarm_rule_trigger_condition_count
  trigger_condition_frequency = var.alarm_rule_trigger_condition_frequency
  send_recovery_notifications = var.alarm_rule_send_recovery_notifications
  recovery_frequency          = var.alarm_rule_send_recovery_notifications ? var.alarm_rule_recovery_frequency : null
  notification_frequency      = var.alarm_rule_notification_frequency
  alarm_rule_alias            = var.alarm_rule_alias

  sql_requests {
    title                  = var.alarm_rule_request_title
    sql                    = var.alarm_rule_request_sql
    log_group_id           = huaweicloud_lts_group.test.id
    log_stream_id          = huaweicloud_lts_stream.test.id
    search_time_range_unit = var.alarm_rule_request_search_time_range_unit
    search_time_range      = var.alarm_rule_request_search_time_range
    log_group_name         = huaweicloud_lts_group.test.group_name
    log_stream_name        = huaweicloud_lts_stream.test.stream_name
  }

  frequency {
    type = var.alarm_rule_frequency_type
  }

  notification_save_rule {
    template_name = var.notification_template_name!= "" ? var.notification_template_name : try([for v in data.huaweicloud_lts_notification_templates.test.templates[*].name :v if v == "sql_template"][0], null)
    user_name     = var.alarm_rule_notification_user_name
    language      = var.alarm_rule_notification_language

    topics {
      name         = huaweicloud_smn_topic.test.name
      topic_urn    = huaweicloud_smn_topic.test.topic_urn
      display_name = huaweicloud_smn_topic.test.display_name
      push_policy  = huaweicloud_smn_topic.test.push_policy
    }
  }
}
```

**Parameter Description**:
- **name**: SQL alarm rule name, assigned by referencing the input variable alarm_rule_name
- **condition_expression**: Alarm condition expression, assigned by referencing the input variable alarm_rule_condition_expression
- **alarm_level**: Alarm level, assigned by referencing the input variable alarm_rule_alarm_level, default value is "MINOR"
- **send_notifications**: Whether to send notifications, set to true to enable notifications
- **trigger_condition_count**: Trigger condition count, assigned by referencing the input variable alarm_rule_trigger_condition_count, default value is 2
- **trigger_condition_frequency**: Trigger condition frequency, assigned by referencing the input variable alarm_rule_trigger_condition_frequency, default value is 3
- **send_recovery_notifications**: Whether to send recovery notifications, assigned by referencing the input variable alarm_rule_send_recovery_notifications, default value is true
- **recovery_frequency**: Recovery frequency, uses alarm_rule_recovery_frequency value when send_recovery_notifications is true
- **notification_frequency**: Notification frequency, assigned by referencing the input variable alarm_rule_notification_frequency, default value is 15
- **alarm_rule_alias**: Alarm rule alias, assigned by referencing the input variable alarm_rule_alias, default value is empty string
- **sql_requests**: SQL request configuration block
  - **title**: Request title, assigned by referencing the input variable alarm_rule_request_title
  - **sql**: SQL query statement, assigned by referencing the input variable alarm_rule_request_sql
  - **log_group_id**: Log group ID, referencing the ID of the previously created log group resource
  - **log_stream_id**: Log stream ID, referencing the ID of the previously created log stream resource
  - **search_time_range_unit**: Search time range unit, assigned by referencing the input variable alarm_rule_request_search_time_range_unit, default value is "minute"
  - **search_time_range**: Search time range, assigned by referencing the input variable alarm_rule_request_search_time_range, default value is 5
  - **log_group_name**: Log group name, referencing the name of the previously created log group resource
  - **log_stream_name**: Log stream name, referencing the name of the previously created log stream resource
- **frequency**: Frequency configuration block
  - **type**: Frequency type, assigned by referencing the input variable alarm_rule_frequency_type, default value is "HOURLY"
- **notification_save_rule**: Notification save rule configuration block
  - **template_name**: Notification template name, prioritizes using notification_template_name variable, if empty then tries to get "sql_template" from query results
  - **user_name**: Notification user name, assigned by referencing the input variable alarm_rule_notification_user_name
  - **language**: Notification language, assigned by referencing the input variable alarm_rule_notification_language, default value is "en-us"
  - **topics**: Topic configuration block
    - **name**: Topic name, referencing the name of the previously created SMN topic resource
    - **topic_urn**: Topic URN, referencing the URN of the previously created SMN topic resource
    - **display_name**: Topic display name, referencing the display name of the previously created SMN topic resource
    - **push_policy**: Push policy, referencing the push policy of the previously created SMN topic resource

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# SMN topic configuration
topic_name = "tf-test-topic"

# Log group configuration
group_name = "tf_test_log_group"

# Log stream configuration
stream_name = "tf_test_log_stream"

# Notification template configuration
domain_id = "your_domain_id"

# SQL alarm rule configuration
alarm_rule_name                   = "tf-test-sql-alarm-rule"
alarm_rule_condition_expression   = "t>0"
alarm_rule_request_title          = "tf-test-sql-alarm-rule-title"
alarm_rule_request_sql            = "select count(*) as t"
alarm_rule_notification_user_name = "your_notification_user_name"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="group_name=my-group" -var="stream_name=my-stream"`
2. Environment variables: `export TF_VAR_group_name=my-group`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating SQL alarm rules
4. Run `terraform show` to view the created SQL alarm rule details

## Reference Information

- [Huawei Cloud Log Tank Service Product Documentation](https://support.huaweicloud.com/lts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [LTS Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/lts)
