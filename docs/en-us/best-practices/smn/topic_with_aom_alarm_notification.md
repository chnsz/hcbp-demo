# Deploy Topic with AOM Alarm Notification

## Application Scenario

Simple Message Notification (SMN) is a reliable and scalable message notification service provided by Huawei Cloud, supporting multiple message notification methods including email, SMS, HTTP/HTTPS, etc. Application Operations Management (AOM) is a one-stop application operations management platform provided by Huawei Cloud, supporting application monitoring, log management, alarm management, and other functions. By configuring AOM alarm notifications to SMN topics, automatic push of alarm messages can be achieved. At the same time, by configuring SMN log tanks, operation logs of SMN topics can be stored in Log Tank Service (LTS), achieving unified log management and analysis. This best practice introduces how to use Terraform to automatically deploy topics with AOM alarm notifications, including creating SMN topics, LTS log groups and streams, SMN log tank configuration, and AOM alarm action rule configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [SMN Topic Resource (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [LTS Log Group Resource (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS Log Stream Resource (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [SMN Log Tank Resource (huaweicloud_smn_logtank)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_logtank)
- [AOM Alarm Action Rule Resource (huaweicloud_aom_alarm_action_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/aom_alarm_action_rule)

### Resource/Data Source Dependencies

```text
huaweicloud_smn_topic
    ├── huaweicloud_smn_logtank
    └── huaweicloud_aom_alarm_action_rule

huaweicloud_lts_group
    └── huaweicloud_lts_stream
        └── huaweicloud_smn_logtank
```

> Note: SMN log tanks depend on SMN topics and LTS log streams, used to store operation logs of SMN topics to LTS. AOM alarm action rules depend on SMN topics, used to configure the sending target of alarm notifications.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create SMN Topic

Add the following script to the TF file (such as main.tf) to create an SMN topic:

```hcl
variable "smn_topic_name" {
  description = "The name of the SMN topic used to send AOM alarm notifications"
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

### 3. Create LTS Log Group and Log Stream

Add the following script to the TF file (such as main.tf) to create LTS log group and log stream:

```hcl
variable "lts_group_name" {
  description = "The name of the LTS group"
  type        = string
}

variable "lts_group_ttl_in_days" {
  description = "The TTL in days of the LTS group"
  type        = number
  default     = 30
}

variable "lts_stream_name" {
  description = "The name of the LTS stream"
  type        = string
}

# Create LTS log group resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_lts_group" "test" {
  group_name            = var.lts_group_name
  ttl_in_days           = var.lts_group_ttl_in_days
  enterprise_project_id = var.enterprise_project_id
}

# Create LTS log stream resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_lts_stream" "test" {
  group_id              = huaweicloud_lts_group.test.id
  stream_name           = var.lts_stream_name
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **group_name**: Log group name, assigned by referencing the input variable `lts_group_name`
- **ttl_in_days**: Log group TTL (days), assigned by referencing the input variable `lts_group_ttl_in_days`, default is 30 days
- **stream_name**: Log stream name, assigned by referencing the input variable `lts_stream_name`
- **group_id**: Log group ID, assigned by referencing the LTS log group resource ID

> Note: Log streams must belong to a log group, so the log group needs to be created first. The log group TTL is used to set the log retention time.

### 4. Configure SMN Log Tank

Add the following script to the TF file (such as main.tf) to configure SMN log tank:

```hcl
# Create SMN log tank resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_smn_logtank" "test" {
  topic_urn     = huaweicloud_smn_topic.test.topic_urn
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id
}
```

**Parameter Description**:
- **topic_urn**: Topic URN, assigned by referencing the SMN topic resource `topic_urn`
- **log_group_id**: Log group ID, assigned by referencing the LTS log group resource ID
- **log_stream_id**: Log stream ID, assigned by referencing the LTS log stream resource ID

> Note: SMN log tanks are used to store operation logs of SMN topics to LTS, achieving unified log management and analysis.

### 5. Configure AOM Alarm Action Rule

Add the following script to the TF file (such as main.tf) to configure AOM alarm action rule:

```hcl
variable "alarm_action_rule_name" {
  description = "The name of the AOM alarm action rule that used to send the SMN notification"
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

variable "alarm_action_rule_description" {
  description = "The description of the AOM alarm action rule"
  type        = string
  default     = null
}

# Create AOM alarm action rule resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_aom_alarm_action_rule" "test" {
  name                  = var.alarm_action_rule_name
  user_name             = var.alarm_action_rule_user_name
  type                  = var.alarm_action_rule_type
  notification_template = "aom.built-in.template.zh"
  description           = var.alarm_action_rule_description

  smn_topics {
    topic_urn = huaweicloud_smn_topic.test.topic_urn
  }
}
```

**Parameter Description**:
- **name**: Alarm action rule name, assigned by referencing the input variable `alarm_action_rule_name`
- **user_name**: User name, assigned by referencing the input variable `alarm_action_rule_user_name`
- **type**: Rule type, assigned by referencing the input variable `alarm_action_rule_type`, default is `1` (notification type)
- **notification_template**: Notification template, set to `aom.built-in.template.zh` indicates using AOM built-in Chinese template
- **description**: Rule description, assigned by referencing the input variable `alarm_action_rule_description`, optional parameter
- **smn_topics**: SMN topic configuration, assigned by referencing the SMN topic resource `topic_urn`, used to specify the sending target of alarm notifications

> Note: AOM alarm action rules are used to configure the sending method of alarm notifications. By configuring SMN topics, automatic push of alarm messages can be achieved. Notification templates can choose built-in templates or custom templates.

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# SMN topic configuration (Required)
smn_topic_name = "tf_test_topic"

# LTS log group and stream configuration (Required)
lts_group_name  = "tf_test_group"
lts_stream_name = "tf_test_stream"

# AOM alarm action rule configuration (Required)
alarm_action_rule_name      = "tf_test_action_rule"
alarm_action_rule_user_name = "your_operation_user_name"
alarm_action_rule_type      = "1"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="smn_topic_name=tf_test_topic" -var="lts_group_name=tf_test_group"`
2. Environment variables: `export TF_VAR_smn_topic_name=tf_test_topic`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating SMN topic, LTS log group and stream, SMN log tank, and AOM alarm action rule
4. Run `terraform show` to view the created resources

## Reference Information

- [Huawei Cloud SMN Product Documentation](https://support.huaweicloud.com/smn/index.html)
- [Huawei Cloud AOM Product Documentation](https://support.huaweicloud.com/aom/index.html)
- [Huawei Cloud LTS Product Documentation](https://support.huaweicloud.com/lts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Topic with AOM Alarm Notification](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/smn/topic-with-aom-alarm-notification)
