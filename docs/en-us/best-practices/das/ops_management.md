# Deploy Ops Management

## Application Scenario

Data Admin Service (DAS) provides intelligent O&M capabilities for DBAs, supporting grouped management of database instances and email template subscription for inspection report notifications. This helps users manage instance health status in batches and receive diagnosis results in a timely manner. By creating instance groups, assigning instances, and configuring email templates with batch subscription, you can automate inspection report notifications and unify O&M. This best practice introduces how to use Terraform to automatically deploy DAS ops management configurations, including creating an instance group, assigning instances to the group, creating an email template, and batch-subscribing to email templates.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Instance Group Resource (huaweicloud_das_instance_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_instance_group)
- [Instance Group Assign Resource (huaweicloud_das_instance_group_assign)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_instance_group_assign)
- [Email Template Resource (huaweicloud_das_email_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_email_template)
- [Email Templates Batch Action Resource (huaweicloud_das_email_templates_batch_action)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_email_templates_batch_action)

### Resource/Data Source Dependencies

```text
huaweicloud_das_instance_group
    ├── huaweicloud_das_instance_group_assign
    └── huaweicloud_das_email_template
        └── huaweicloud_das_email_templates_batch_action
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the configuration introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create Instance Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an instance group resource:

```hcl
variable "ops_datastore_type" {
  description = "The database type"
  type        = string
}

variable "ops_group_name" {
  description = "The instance group name"
  type        = string
}

variable "ops_group_description" {
  description = "The description of the instance group"
  type        = string
}

# Create an instance group resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_instance_group" "test" {
  datastore_type = var.ops_datastore_type
  group_name     = var.ops_group_name
  description    = var.ops_group_description
}
```

**Parameter Description**:
- **datastore_type**: Database type, assigned by referencing the input variable `ops_datastore_type`
- **group_name**: Instance group name, assigned by referencing the input variable `ops_group_name`
- **description**: Description of the instance group, assigned by referencing the input variable `ops_group_description`

### 3. Assign Instances to the Instance Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an instance group assign resource:

```hcl
variable "ops_group_instance_ids" {
  description = "The list of instance IDs to be assigned to the group"
  type        = list(string)
}

# Create an instance group assign resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_instance_group_assign" "test" {
  group_id     = huaweicloud_das_instance_group.test.id
  instance_ids = var.ops_group_instance_ids
}
```

**Parameter Description**:
- **group_id**: Instance group ID, assigned by referencing the ID of the instance group resource
- **instance_ids**: List of instance IDs to be assigned to the group, assigned by referencing the input variable `ops_group_instance_ids`

> Note: Ensure that the target database instances already exist and that the instance type matches the datastore type of the instance group before assignment.

### 4. Create Email Template

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an email template resource:

```hcl
variable "ops_email_template_name" {
  description = "The name of the email template"
  type        = string
}

variable "ops_email_health_rank" {
  description = "The list of health ranks"
  type        = list(string)
}

variable "ops_email_inspection_time" {
  description = "The diagnosis time"
  type        = string
}

variable "ops_email_send_time" {
  description = "The send time"
  type        = string
}

variable "ops_email_time_zone" {
  description = "The time zone"
  type        = string
}

variable "ops_email_address" {
  description = "The email address for notification"
  type        = string
  default     = null
}

variable "ops_email_topic" {
  description = "The topic ID for notification"
  type        = string
  default     = null
}

variable "ops_email_topic_urn" {
  description = "The topic URN for notification"
  type        = string
  default     = null
}

variable "ops_email_obs_bucket_name" {
  description = "The OBS bucket name for storing inspection reports"
  type        = string
  default     = null
}

# Create an email template resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_email_template" "test" {
  datastore_type  = var.ops_datastore_type
  name            = var.ops_email_template_name
  groups          = [huaweicloud_das_instance_group.test.id]
  health_rank     = var.ops_email_health_rank
  inspection_time = var.ops_email_inspection_time
  send_time       = var.ops_email_send_time
  time_zone       = var.ops_email_time_zone
  email           = var.ops_email_address
  topic           = var.ops_email_topic
  topic_urn       = var.ops_email_topic_urn
  obs_bucket_name = var.ops_email_obs_bucket_name
}
```

**Parameter Description**:
- **datastore_type**: Database type, assigned by referencing the input variable `ops_datastore_type`
- **name**: Name of the email template, assigned by referencing the input variable `ops_email_template_name`
- **groups**: List of associated instance group IDs, assigned by referencing the ID of the instance group resource
- **health_rank**: List of health ranks, assigned by referencing the input variable `ops_email_health_rank`
- **inspection_time**: Diagnosis time, assigned by referencing the input variable `ops_email_inspection_time`
- **send_time**: Send time, assigned by referencing the input variable `ops_email_send_time`
- **time_zone**: Time zone, assigned by referencing the input variable `ops_email_time_zone`
- **email**: Email address for notification, assigned by referencing the input variable `ops_email_address`
- **topic**: Topic ID for notification, assigned by referencing the input variable `ops_email_topic`
- **topic_urn**: Topic URN for notification, assigned by referencing the input variable `ops_email_topic_urn`
- **obs_bucket_name**: OBS bucket name for storing inspection reports, assigned by referencing the input variable `ops_email_obs_bucket_name`

> Note: The email template depends on an existing instance group. Configure email, SMN topic, or OBS bucket parameters for notification and storage as needed.

### 5. Batch Subscribe to Email Templates

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an email templates batch action resource:

```hcl
variable "ops_email_subscribe" {
  description = "Whether to subscribe to the email templates"
  type        = bool
}

# Create an email templates batch action resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_email_templates_batch_action" "test" {
  subscribe          = var.ops_email_subscribe
  email_template_ids = [huaweicloud_das_email_template.test.id]
}
```

**Parameter Description**:
- **subscribe**: Whether to subscribe to the email templates, assigned by referencing the input variable `ops_email_subscribe`
- **email_template_ids**: List of email template IDs for the batch action, assigned by referencing the ID of the email template resource

> Note: Batch subscription depends on an existing email template.

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables for configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory. Example content is as follows:

```hcl
# Instance group configuration
ops_datastore_type     = "MySQL"
ops_group_name         = "tf_test_das_group"
ops_group_description  = "tf_test_das_group_description"
ops_group_instance_ids = ["your_instance_id_1", "your_instance_id_2"]

# Email template configuration
ops_email_template_name   = "tf_test_email_template"
ops_email_health_rank     = ["dangerous", "sub_healthy"]
ops_email_inspection_time = "00:00-00:00"
ops_email_send_time       = "08:00-10:00"
ops_email_time_zone       = "Asia/Shanghai"
ops_email_address         = "tf_test@example.com"
ops_email_obs_bucket_name = "your bucket name"

# Email template batch subscription configuration
ops_email_subscribe = true
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; for other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="ops_group_name=tf_test_das_group" -var="ops_email_subscribe=true"`
2. Environment variables: `export TF_VAR_ops_group_name=tf_test_das_group`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the ops management configurations
4. Run `terraform show` to view the created ops management configurations

## Reference Information

- [Huawei Cloud DAS Product Documentation](https://support.huaweicloud.com/das/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DAS Ops Management Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/das/ops-management)
