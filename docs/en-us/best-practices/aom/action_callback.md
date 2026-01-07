# Deploy AOM Alarm Action Callback

## Application Scenario

Application Operations Management (AOM) is a one-stop application operations management platform provided by Huawei Cloud, supporting core functions such as application monitoring, log management, and alarm management. By configuring AOM alarm action callbacks, alarm information can be sent to specified callback URLs through Simple Message Notification (SMN), enabling real-time notification and processing of alarm information. Alarm action callbacks help users quickly respond to alarm events and improve operational efficiency.

This best practice will introduce how to use Terraform to automatically deploy AOM alarm action callbacks, including creating SMN topics and subscriptions, AOM message templates, and configuring AOM alarm action rules.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Simple Message Notification Topic Resource (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [Simple Message Notification Subscription Resource (huaweicloud_smn_subscription)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_subscription)
- [AOM Message Template Resource (huaweicloud_aom_message_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_message_template)
- [AOM Alarm Action Rule Resource (huaweicloud_aom_alarm_action_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_alarm_action_rule)

### Resource/Data Source Dependencies

```
huaweicloud_smn_topic
    └── huaweicloud_smn_subscription

huaweicloud_aom_message_template
    └── huaweicloud_aom_alarm_action_rule

huaweicloud_smn_topic
    └── huaweicloud_aom_alarm_action_rule
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Simple Message Notification Topic Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Simple Message Notification topic resource:

```hcl
variable "smn_topic_name" {
  description = "The name of the SMN topic that used to send the SMN notification"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
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

### 3. Create Simple Message Notification Subscription Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Simple Message Notification subscription resource:

```hcl
variable "alarm_callback_urls" {
  description = "The URLs of the alarm callback"
  type        = list(string)

  validation {
    condition     = length(var.alarm_callback_urls) > 0 && alltrue([for url in var.alarm_callback_urls : length(regexall("^http[s]?://", url)) > 0])
    error_message = "The alarm callback URLs must be provided and must start with http:// or https://"
  }
}

# Create Simple Message Notification subscription resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_smn_subscription" "test" {
  count = length(var.alarm_callback_urls) > 0 ? 1 : 0

  topic_urn = huaweicloud_smn_topic.test.id
  protocol  = length(regexall("^https?://", var.alarm_callback_urls[count.index])) > 0 ? "https" : "http"
  endpoint  = var.alarm_callback_urls[count.index]
}
```

**Parameter Description**:
- **count**: The number of subscription resources to create, used to control whether to create subscriptions, only created when the alarm_callback_urls list is not empty
- **topic_urn**: The topic URN, referencing the ID of the previously created Simple Message Notification topic resource (huaweicloud_smn_topic.test)
- **protocol**: The protocol of the message receiving endpoint, automatically determined as http or https based on the callback URL prefix
- **endpoint**: The message receiving endpoint address, assigned by referencing elements in the input variable alarm_callback_urls list

> Note: In actual use, if alarm_callback_urls contains multiple URLs, separate subscription resources need to be created for each URL. The above example code needs to be adjusted according to actual requirements.

### 4. Create AOM Message Template Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an AOM message template resource:

```hcl
variable "alarm_notification_template_name" {
  description = "The name of the AOM alarm notification template that used to send the SMN notification"
  type        = string
}

variable "alarm_notification_template_locale" {
  description = "The locale of the AOM alarm notification template that used to send the SMN notification"
  type        = string
  default     = "en-us"

  validation {
    condition     = contains(["en-us", "zh-cn"], var.alarm_notification_template_locale)
    error_message = "The alarm notification template locale must be 'en-us' or 'zh-cn'"
  }
}

variable "alarm_notification_template_description" {
  description = "The description of the AOM alarm notification template that used to send the SMN notification"
  type        = string
  default     = ""
}

variable "alarm_notification_template_notification_type" {
  description = "The notification type of the AOM alarm notification template that used to send the SMN notification"
  type        = string
  default     = "email"
}

variable "alarm_notification_template_notification_topic" {
  description = "The notification topic of the AOM alarm notification template that used to send the SMN notification"
  type        = string
  default     = "An alert occurred at time $${starts_at}[$${event_severity}_$${event_type}_$${clear_type}]."
}

variable "alarm_notification_template_content" {
  description = "The content of the AOM alarm notification template that used to send the SMN notification"
  type        = string
  default     = <<EOT
Alarm Name: $${event_name_alias};
Alarm ID: $${id};
Notification Rule: $${action_rule};
Trigger Time: $${starts_at};
Trigger Level: $${event_severity};
Alarm Content: $${alarm_info};
Resource Identifier: $${resources_new};
Remediation Suggestion: $${alarm_fix_suggestion_zh};
  EOT
}

# Create AOM message template resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_aom_message_template" "test" {
  name                  = var.alarm_notification_template_name
  locale                = var.alarm_notification_template_locale
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  description           = var.alarm_notification_template_description

  templates {
    sub_type = var.alarm_notification_template_notification_type
    topic    = var.alarm_notification_template_notification_topic
    content  = var.alarm_notification_template_content
  }
}
```

**Parameter Description**:
- **name**: The message template name, assigned by referencing the input variable alarm_notification_template_name
- **locale**: The message template language, assigned by referencing the input variable alarm_notification_template_locale, default value is "en-us", valid values are "en-us" or "zh-cn"
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, set to null when the value is an empty string
- **description**: The message template description, assigned by referencing the input variable alarm_notification_template_description, default value is an empty string
- **templates.sub_type**: The notification type, assigned by referencing the input variable alarm_notification_template_notification_type, default value is "email"
- **templates.topic**: The notification topic template, assigned by referencing the input variable alarm_notification_template_notification_topic, supports variable substitution
- **templates.content**: The notification content template, assigned by referencing the input variable alarm_notification_template_content, supports variable substitution

### 5. Create AOM Alarm Action Rule Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an AOM alarm action rule resource:

```hcl
variable "alarm_action_rule_name" {
  description = "The name of the AOM alarm action rule that used to send the SMN notification"
  type        = string
}

variable "alarm_action_rule_user_name" {
  description = "The user name of the AOM alarm action rule that used to send the SMN notification"
  type        = string
}

variable "alarm_action_rule_type" {
  description = "The type of the AOM alarm action rule that used to send the SMN notification"
  type        = string
  default     = "1" # notification
}

# Create AOM alarm action rule resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_aom_alarm_action_rule" "test" {
  depends_on = [huaweicloud_aom_message_template.test]

  name                  = var.alarm_action_rule_name
  user_name             = var.alarm_action_rule_user_name
  type                  = var.alarm_action_rule_type
  notification_template = huaweicloud_aom_message_template.test.name

  smn_topics {
    topic_urn = huaweicloud_smn_topic.test.topic_urn
  }
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency relationship, ensuring the AOM message template resource is created before the alarm action rule resource
- **name**: The alarm action rule name, assigned by referencing the input variable alarm_action_rule_name
- **user_name**: The user name, assigned by referencing the input variable alarm_action_rule_user_name
- **type**: The alarm action rule type, assigned by referencing the input variable alarm_action_rule_type, default value is "1" (indicating notification type)
- **notification_template**: The notification template name, referencing the name of the previously created AOM message template resource (huaweicloud_aom_message_template.test)
- **smn_topics.topic_urn**: The SMN topic URN, referencing the topic_urn of the previously created Simple Message Notification topic resource (huaweicloud_smn_topic.test)

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Simple Message Notification Topic Configuration
smn_topic_name = "tf_test_alarm_action_callback"

# Alarm Callback URL Configuration
alarm_callback_urls = ["https://www.example.com"]

# AOM Message Template Configuration
alarm_notification_template_name        = "tf_test_alarm_action_callback"
alarm_notification_template_description = "This is a AOM alarm notification template created by Terraform"

# AOM Alarm Action Rule Configuration
alarm_action_rule_name      = "tf_test_alarm_action_callback"
alarm_action_rule_user_name = "your_operation_user_name"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="smn_topic_name=test-topic" -var="alarm_callback_urls=[\"https://www.example.com\"]"`
2. Environment variables: `export TF_VAR_smn_topic_name=test-topic`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the AOM alarm action callback
4. Run `terraform show` to view the details of the created AOM alarm action callback

## Reference Information

- [Huawei Cloud AOM Product Documentation](https://support.huaweicloud.com/aom/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For AOM Alarm Action Callback](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/aom/action-callback)
