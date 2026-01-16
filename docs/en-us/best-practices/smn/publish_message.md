# Deploy Message Publish

## Application Scenario

Simple Message Notification (SMN) is a reliable and scalable message notification service provided by Huawei Cloud, supporting multiple message notification methods including email, SMS, HTTP/HTTPS, etc. Through message publishing, messages can be published to SMN topics, and terminals subscribed to the topic will receive message notifications. Message publishing supports multiple methods including direct message content, message structure, and message templates, meeting message publishing requirements for different scenarios. This best practice introduces how to use Terraform to automatically deploy message publishing, including creating SMN topics, subscriptions, message templates (optional), and publishing messages.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [SMN Topic Resource (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [SMN Subscription Resource (huaweicloud_smn_subscription)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_subscription)
- [SMN Message Template Resource (huaweicloud_smn_message_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_message_template)
- [SMN Message Publish Resource (huaweicloud_smn_message_publish)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_message_publish)

### Resource/Data Source Dependencies

```text
huaweicloud_smn_topic
    ├── huaweicloud_smn_subscription
    └── huaweicloud_smn_message_publish

huaweicloud_smn_message_template
    └── huaweicloud_smn_message_publish
```

> Note: Message publishing depends on SMN topics, and message templates can be optionally used. Subscriptions are used to receive message notifications. After messages are published, terminals subscribed to the topic will receive messages.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create SMN Topic

Add the following script to the TF file (such as main.tf) to create an SMN topic:

```hcl
variable "topic_name" {
  description = "The name of the SMN topic"
  type        = string
}

variable "topic_display_name" {
  description = "The display name of the SMN topic"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = null
}

# Create SMN topic resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_smn_topic" "test" {
  name                  = var.topic_name
  display_name          = var.topic_display_name
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **name**: Topic name, assigned by referencing the input variable `topic_name`
- **display_name**: Topic display name, assigned by referencing the input variable `topic_display_name`, optional parameter
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable `enterprise_project_id`, optional parameter

### 3. Create SMN Subscription

Add the following script to the TF file (such as main.tf) to create an SMN subscription:

```hcl
variable "subscription_protocol" {
  description = "The protocol of the subscription"
  type        = string
}

variable "subscription_endpoint" {
  description = "The endpoint of the subscription"
  type        = string
}

variable "subscription_description" {
  description = "The remark for SMN subscriptions"
  type        = string
  default     = ""
}

# Create SMN subscription resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_smn_subscription" "test" {
  topic_urn = huaweicloud_smn_topic.test.id
  protocol  = var.subscription_protocol
  endpoint  = var.subscription_endpoint
  remark    = var.subscription_description
}
```

**Parameter Description**:
- **topic_urn**: Topic URN, assigned by referencing the SMN topic resource ID
- **protocol**: Subscription protocol, assigned by referencing the input variable `subscription_protocol`. Supports `email`, `sms`, `http`, `https`, `functionstage`, etc.
- **endpoint**: Subscription endpoint, assigned by referencing the input variable `subscription_endpoint`. Fill in the corresponding endpoint address according to the protocol type
- **remark**: Subscription remark, assigned by referencing the input variable `subscription_description`, optional parameter

> Note: Subscription protocol and endpoint must match. For example, SMS protocol requires a phone number, and email protocol requires an email address.

### 4. Create Message Template (Optional)

Add the following script to the TF file (such as main.tf) to create a message template (optional):

```hcl
variable "template_name" {
  description = "The name of the message template"
  type        = string
  default     = ""
  nullable    = false
}

variable "template_protocol" {
  description = "The protocol of the message template"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.template_name == "" || var.template_protocol != ""
    error_message = "The template_protocol is required if template_name is provided."
  }
}

variable "template_content" {
  description = "The content of the message template"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.template_name == "" || var.template_content != ""
    error_message = "The template_content is required if template_name is provided."
  }
}

# Create message template (create when template_name is not empty)
resource "huaweicloud_smn_message_template" "test" {
  count    = var.template_name != "" ? 1 : 0

  name     = var.template_name
  protocol = var.template_protocol
  content  = var.template_content
}
```

**Parameter Description**:
- **count**: Resource count, creates resource when `template_name` is not empty
- **name**: Template name, assigned by referencing the input variable `template_name`
- **protocol**: Template protocol, assigned by referencing the input variable `template_protocol`
- **content**: Template content, assigned by referencing the input variable `template_content`

> Note: Message templates are optional. If using templates to publish messages, message templates need to be created first. Template protocol must match subscription protocol.

### 5. Publish Message

Add the following script to the TF file (such as main.tf) to publish a message:

```hcl
variable "pulblish_subject" {
  description = "The subject of the message"
  type        = string
}

variable "pulblish_message" {
  description = "The message content (mutually exclusive with message_structure)"
  type        = string
  default     = ""
  nullable    = false
}

variable "pulblish_message_structure" {
  description = "The JSON message structure that allows sending different content to different protocol subscribers (mutually exclusive with message)"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = !(var.template_name == "" && var.pulblish_message == "") || var.pulblish_message_structure != ""
    error_message = "The pulblish_message_structure is required if both template_name and pulblish_message are not provided."
  }
}

variable "pulblish_time_to_live" {
  description = "The maximum retention time of the message within the SMN system in seconds (default: 3600, max: 86400)"
  type        = string
  default     = null
}

variable "pulblish_tags" {
  description = "The tags of the message"
  type        = map(string)
  default     = {}
}

variable "pulblish_message_attributes" {
  description = "The message attributes of the message"
  type        = list(object({
    name   = string
    type   = string
    value  = optional(string)
    values = optional(list(string))
  }))

  default  = []
  nullable = false
}

# Publish message in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_smn_message_publish" "test" {
  topic_urn             = huaweicloud_smn_topic.test.topic_urn
  subject               = var.pulblish_subject
  message               = var.pulblish_message != "" ? var.pulblish_message : null
  message_structure     = var.pulblish_message_structure != "" ? var.pulblish_message_structure : null
  message_template_name = var.template_name != "" ? try(huaweicloud_smn_message_template.test[0].name, null) : null
  time_to_live          = var.pulblish_time_to_live
  tags                  = var.pulblish_tags

  dynamic "message_attributes" {
    for_each = var.pulblish_message_attributes

    content {
      name   = message_attributes.value.name
      type   = message_attributes.value.type
      value  = message_attributes.value.value
      values = message_attributes.value.values
    }
  }
}
```

**Parameter Description**:
- **topic_urn**: Topic URN, assigned by referencing the SMN topic resource `topic_urn`
- **subject**: Message subject, assigned by referencing the input variable `pulblish_subject`
- **message**: Message content, assigned by referencing the input variable `pulblish_message`, mutually exclusive with `message_structure`, optional parameter
- **message_structure**: Message structure, assigned by referencing the input variable `pulblish_message_structure`, JSON format, allows sending different content to different protocol subscribers, mutually exclusive with `message`, optional parameter
- **message_template_name**: Message template name, references message template resource name when `template_name` is not empty, optional parameter
- **time_to_live**: Maximum retention time of the message within the SMN system (seconds), assigned by referencing the input variable `pulblish_time_to_live`, default is 3600 seconds, maximum 86400 seconds, optional parameter
- **tags**: Message tags, assigned by referencing the input variable `pulblish_tags`, optional parameter
- **message_attributes**: Message attribute list, creates multiple message attributes through dynamic block `dynamic "message_attributes"` based on input variable `pulblish_message_attributes`, optional parameter

> Note: Message publishing supports three methods: direct message content (message), message structure (message_structure), and message template (message_template_name). Message and message structure are mutually exclusive. If using message templates, message content is not required. Message structure is in JSON format and can send different content to subscribers of different protocols.

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# SMN topic configuration (Required)
topic_name = "tf_test_topic"

# SMN subscription configuration (Required)
subscription_protocol      = "sms"
subscription_endpoint      = "18629199536"

# Message publishing configuration (Required)
pulblish_subject           = "tf_test_subject"
pulblish_message_structure = "{\"default\":\"Dear user, this is a default message.\",\"sms\":\"Dear user, this is an SMS message.\"}"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="topic_name=tf_test_topic" -var="subscription_protocol=sms"`
2. Environment variables: `export TF_VAR_topic_name=tf_test_topic`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating SMN topic, subscription, and publishing messages
4. Run `terraform show` to view the created resources

## Reference Information

- [Huawei Cloud SMN Product Documentation](https://support.huaweicloud.com/smn/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Message Publish](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/smn/publish-message)
