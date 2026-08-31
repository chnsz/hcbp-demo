# Deploy Lock Analysis

## Application Scenario

Data Admin Service (DAS) provides intelligent database O&M capabilities, supporting deadlock detection and historical transaction analysis for database instances to help DBAs quickly locate lock waits, deadlocks, and long-running transactions. By enabling full deadlock detection and the historical transaction switch, and exporting historical transactions to an OBS bucket as needed, you can achieve continuous observation and offline analysis of lock-related issues. This best practice introduces how to use Terraform to automatically deploy DAS lock analysis configurations, including enabling full deadlock detection, enabling the historical transaction switch, and creating a historical transaction export task.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Full Dead Lock Switch Resource (huaweicloud_das_full_dead_lock_switch)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_full_dead_lock_switch)
- [History Transaction Switch Resource (huaweicloud_das_history_transaction_switch)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_history_transaction_switch)
- [History Transaction Export Task Resource (huaweicloud_das_history_transaction_export_task)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_history_transaction_export_task)

### Resource/Data Source Dependencies

```text
huaweicloud_das_full_dead_lock_switch

huaweicloud_das_history_transaction_switch
    └── huaweicloud_das_history_transaction_export_task
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the configuration introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Enable Full Dead Lock Detection

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a full dead lock switch resource:

```hcl
variable "lock_analysis_instance_id" {
  description = "The ID of the database instances, separated by commas"
  type        = string
}

variable "full_dead_lock_switch_on" {
  description = "Whether to enable the full dead lock switch"
  type        = bool
  default     = false
}

variable "full_dead_lock_retention_hours" {
  description = "The retention hours of the full dead lock data"
  type        = number
  default     = null
}

# Create a full dead lock switch resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_full_dead_lock_switch" "test" {
  instance_id = var.lock_analysis_instance_id

  switch_on = var.full_dead_lock_switch_on

  retention_hours = var.full_dead_lock_retention_hours
}
```

**Parameter Description**:
- **instance_id**: ID of the database instances (separated by commas), assigned by referencing the input variable `lock_analysis_instance_id`
- **switch_on**: Whether to enable the full dead lock switch, assigned by referencing the input variable `full_dead_lock_switch_on`
- **retention_hours**: Retention hours of the full dead lock data, assigned by referencing the input variable `full_dead_lock_retention_hours`

> Note: A target database instance must already exist before deployment. Configure the deadlock data retention period reasonably based on business requirements.

### 3. Enable History Transaction Switch

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a history transaction switch resource:

```hcl
variable "history_transaction_status" {
  description = "The switch status of the history transaction"
  type        = string
  default     = "Enabled"
}

variable "lock_analysis_datastore_type" {
  description = "The database type"
  type        = string
  default     = "MySQL"
}

# Create a history transaction switch resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_history_transaction_switch" "test" {
  instance_id    = var.lock_analysis_instance_id
  status         = var.history_transaction_status
  datastore_type = var.lock_analysis_datastore_type
}
```

**Parameter Description**:
- **instance_id**: ID of the database instances (separated by commas), assigned by referencing the input variable `lock_analysis_instance_id`
- **status**: Switch status of the history transaction, assigned by referencing the input variable `history_transaction_status`
- **datastore_type**: Database type, assigned by referencing the input variable `lock_analysis_datastore_type`

### 4. Create History Transaction Export Task

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a history transaction export task resource:

```hcl
variable "history_transaction_bucket_name" {
  description = "The OBS bucket name for exporting history transactions"
  type        = string
}

variable "history_transaction_start_time" {
  description = "The start time of the history transactions to export, in RFC3339 format"
  type        = string
  default     = "2000-06-01T00:00:00+08:00"
}

variable "history_transaction_end_time" {
  description = "The end time of the history transactions to export, in RFC3339 format"
  type        = string
  default     = "2099-06-02T00:00:00+08:00"
}

variable "history_transaction_file_path" {
  description = "The OBS file directory for the export task"
  type        = string
  default     = null
}

variable "history_transaction_time_zone" {
  description = "The time zone for the export task"
  type        = string
  default     = "UTC+8"
}

variable "history_transaction_order_field" {
  description = "The sort field for the export task"
  type        = string
  default     = "collectTime"
}

variable "history_transaction_order_by" {
  description = "The sort order for the export task"
  type        = string
  default     = "asc"
}

variable "history_transaction_last_sec_min" {
  description = "The minimum duration for the export task"
  type        = number
  default     = 0
}

variable "history_transaction_last_sec_max" {
  description = "The maximum duration for the export task"
  type        = number
  default     = 100
}

# Create a history transaction export task resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_history_transaction_export_task" "test" {
  instance_id = var.lock_analysis_instance_id
  bucket_name = var.history_transaction_bucket_name
  start_time  = var.history_transaction_start_time
  end_time    = var.history_transaction_end_time
  file_path   = var.history_transaction_file_path
  time_zone   = var.history_transaction_time_zone
  order_field = var.history_transaction_order_field
  order_by    = var.history_transaction_order_by

  last_sec_min = var.history_transaction_last_sec_min
  last_sec_max = var.history_transaction_last_sec_max

  depends_on = [huaweicloud_das_history_transaction_switch.test]
}
```

**Parameter Description**:
- **instance_id**: ID of the database instances (separated by commas), assigned by referencing the input variable `lock_analysis_instance_id`
- **bucket_name**: OBS bucket name for exporting history transactions, assigned by referencing the input variable `history_transaction_bucket_name`
- **start_time**: Start time of the history transactions to export in RFC3339 format, assigned by referencing the input variable `history_transaction_start_time`
- **end_time**: End time of the history transactions to export in RFC3339 format, assigned by referencing the input variable `history_transaction_end_time`
- **file_path**: OBS file directory for the export task, assigned by referencing the input variable `history_transaction_file_path`
- **time_zone**: Time zone for the export task, assigned by referencing the input variable `history_transaction_time_zone`
- **order_field**: Sort field for the export task, assigned by referencing the input variable `history_transaction_order_field`
- **order_by**: Sort order for the export task, assigned by referencing the input variable `history_transaction_order_by`
- **last_sec_min**: Minimum duration for the export task, assigned by referencing the input variable `history_transaction_last_sec_min`
- **last_sec_max**: Maximum duration for the export task, assigned by referencing the input variable `history_transaction_last_sec_max`

> Note: The history transaction export task depends on an enabled history transaction switch. Ensure that the OBS bucket already exists and that you have the required write permissions.

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables for configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory. Example content is as follows:

```hcl
# Lock analysis instance configuration
lock_analysis_instance_id = "your_rds_instance_id"

# Full dead lock detection configuration
full_dead_lock_switch_on       = true
full_dead_lock_retention_hours = 24

# History transaction switch configuration
history_transaction_status   = "Enabled"
lock_analysis_datastore_type = "MySQL"

# History transaction export task configuration
history_transaction_bucket_name = "your_obs_bucket"
history_transaction_start_time  = "2026-08-01T00:00:00Z"
history_transaction_end_time    = "2026-08-10T23:59:59Z"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; for other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="lock_analysis_instance_id=your_rds_instance_id" -var="history_transaction_bucket_name=your_obs_bucket"`
2. Environment variables: `export TF_VAR_lock_analysis_instance_id=your_rds_instance_id`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the lock analysis configurations
4. Run `terraform show` to view the created lock analysis configurations

## Reference Information

- [Huawei Cloud DAS Product Documentation](https://support.huaweicloud.com/das/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DAS Lock Analysis Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/das/lock-analysis)
