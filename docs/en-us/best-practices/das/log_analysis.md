# Deploy Log Analysis

## Application Scenario

Data Admin Service (DAS) provides database log analysis capabilities, supporting slow log export and Binlog parsing to help developers and DBAs locate slow SQL statements, trace data changes, and troubleshoot issues. By exporting slow logs and Binlog parsing results to an OBS bucket, you can perform offline analysis and long-term retention of log data. This best practice introduces how to use Terraform to automatically deploy DAS log analysis configurations, including querying database users, creating a slow log export task, creating a Binlog parse task, and exporting Binlog parse results.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Database Users Data Source (data.huaweicloud_das_database_users)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/das_database_users)

### Resources

- [Slow Log Export Task Resource (huaweicloud_das_slow_log_export_task)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_slow_log_export_task)
- [Binlog Parse Task Resource (huaweicloud_das_binlog_parse_task)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_binlog_parse_task)
- [Binlog Parse Task Export Resource (huaweicloud_das_binlog_parse_task_export)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_binlog_parse_task_export)

### Resource/Data Source Dependencies

```text
data.huaweicloud_das_database_users
    └── huaweicloud_das_binlog_parse_task
        └── huaweicloud_das_binlog_parse_task_export

huaweicloud_das_slow_log_export_task
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the configuration introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Database Users

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the Binlog parse task:

```hcl
variable "log_analysis_instance_id" {
  description = "The ID of the database instance"
  type        = string
  default     = ""
}

# Query the database user list of the specified database instance in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the Binlog parse task
data "huaweicloud_das_database_users" "test" {
  instance_id = var.log_analysis_instance_id
}

locals {
  user_id = try(data.huaweicloud_das_database_users.test.users[0].id, null)
}
```

**Parameter Description**:
- **instance_id**: ID of the database instance, assigned by referencing the input variable `log_analysis_instance_id`

### 3. Create Slow Log Export Task

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a slow log export task resource:

```hcl
variable "slow_log_bucket_name" {
  description = "The OBS bucket name for exporting slow logs"
  type        = string
}

variable "slow_log_start_time" {
  description = "The start time of the slow logs to export, in RFC3339 format"
  type        = string
}

variable "slow_log_end_time" {
  description = "The end time of the slow logs to export, in RFC3339 format"
  type        = string
}

variable "slow_log_file_path" {
  description = "The OBS file directory for the export task"
  type        = string
  default     = null
}

variable "slow_log_export_type" {
  description = "The export type for the slow log export task"
  type        = string
  default     = null
}

variable "slow_log_sort_field" {
  description = "The sort field for the slow log export task"
  type        = string
  default     = null
}

variable "slow_log_sort_asc" {
  description = "Whether to sort in ascending order"
  type        = bool
  default     = null
}

variable "slow_log_time_zone" {
  description = "The time zone for the slow log export task"
  type        = string
  default     = null
}

# Create a slow log export task resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_slow_log_export_task" "test" {
  instance_id = var.log_analysis_instance_id
  bucket_name = var.slow_log_bucket_name
  start_time  = var.slow_log_start_time
  end_time    = var.slow_log_end_time
  file_path   = var.slow_log_file_path
  export_type = var.slow_log_export_type
  sort_field  = var.slow_log_sort_field
  sort_asc    = var.slow_log_sort_asc
  time_zone   = var.slow_log_time_zone
}
```

**Parameter Description**:
- **instance_id**: ID of the database instance, assigned by referencing the input variable `log_analysis_instance_id`
- **bucket_name**: OBS bucket name for exporting slow logs, assigned by referencing the input variable `slow_log_bucket_name`
- **start_time**: Start time of the slow logs to export in RFC3339 format, assigned by referencing the input variable `slow_log_start_time`
- **end_time**: End time of the slow logs to export in RFC3339 format, assigned by referencing the input variable `slow_log_end_time`
- **file_path**: OBS file directory for the export task, assigned by referencing the input variable `slow_log_file_path`
- **export_type**: Export type for the slow log export task, assigned by referencing the input variable `slow_log_export_type`
- **sort_field**: Sort field for the slow log export task, assigned by referencing the input variable `slow_log_sort_field`
- **sort_asc**: Whether to sort in ascending order, assigned by referencing the input variable `slow_log_sort_asc`
- **time_zone**: Time zone for the slow log export task, assigned by referencing the input variable `slow_log_time_zone`

> Note: Ensure that the OBS bucket already exists and that you have the required write permissions.

### 4. Create Binlog Parse Task

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Binlog parse task resource:

```hcl
variable "binlog_binlog_type" {
  description = "The binlog type"
  type        = string
}

variable "binlog_file_name" {
  description = "The binlog file name"
  type        = string
}

variable "binlog_backup_id" {
  description = "The backup ID"
  type        = string
  default     = null
}

# Create a Binlog parse task resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_binlog_parse_task" "test" {
  user_id     = local.user_id
  binlog_type = var.binlog_binlog_type
  file_name   = var.binlog_file_name
  backup_id   = var.binlog_backup_id
}
```

**Parameter Description**:
- **user_id**: Database user ID, assigned based on the result of the database users data source (`data.huaweicloud_das_database_users`)
- **binlog_type**: Binlog type, assigned by referencing the input variable `binlog_binlog_type`
- **file_name**: Binlog file name, assigned by referencing the input variable `binlog_file_name`
- **backup_id**: Backup ID, assigned by referencing the input variable `binlog_backup_id`

### 5. Export Binlog Parse Results

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Binlog parse task export resource:

```hcl
variable "binlog_export_bucket_name" {
  description = "The OBS bucket name for exporting binlog parse results"
  type        = string
}

variable "binlog_filter_db_names" {
  description = "The list of database names to filter"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "binlog_filter_tb_names" {
  description = "The list of table names to filter"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "binlog_filter_start_time" {
  description = "The start time of the export range, in RFC3339 format"
  type        = string
  default     = null
}

variable "binlog_filter_end_time" {
  description = "The end time of the export range, in RFC3339 format"
  type        = string
  default     = null
}

variable "binlog_filter_types" {
  description = "The list of SQL types to filter"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "binlog_filter_parse_double_insert" {
  description = "Whether to export UPDATE statements as two INSERT statements"
  type        = bool
  default     = null
}

# Create a Binlog parse task export resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_binlog_parse_task_export" "test" {
  user_id     = local.user_id
  task_id     = huaweicloud_das_binlog_parse_task.test.id
  bucket_name = var.binlog_export_bucket_name

  filter_condition {
    db_names            = var.binlog_filter_db_names
    tb_names            = var.binlog_filter_tb_names
    start_time          = var.binlog_filter_start_time
    end_time            = var.binlog_filter_end_time
    types               = var.binlog_filter_types
    parse_double_insert = var.binlog_filter_parse_double_insert
  }
}
```

**Parameter Description**:
- **user_id**: Database user ID, assigned based on the result of the database users data source (`data.huaweicloud_das_database_users`)
- **task_id**: Binlog parse task ID, assigned by referencing the ID of the Binlog parse task resource
- **bucket_name**: OBS bucket name for exporting Binlog parse results, assigned by referencing the input variable `binlog_export_bucket_name`
- **filter_condition**: Export filter condition configuration
  - **db_names**: List of database names to filter, assigned by referencing the input variable `binlog_filter_db_names`
  - **tb_names**: List of table names to filter, assigned by referencing the input variable `binlog_filter_tb_names`
  - **start_time**: Start time of the export range in RFC3339 format, assigned by referencing the input variable `binlog_filter_start_time`
  - **end_time**: End time of the export range in RFC3339 format, assigned by referencing the input variable `binlog_filter_end_time`
  - **types**: List of SQL types to filter, assigned by referencing the input variable `binlog_filter_types`
  - **parse_double_insert**: Whether to export UPDATE statements as two INSERT statements, assigned by referencing the input variable `binlog_filter_parse_double_insert`

> Note: The Binlog parse result export depends on an existing Binlog parse task. Ensure that the OBS bucket already exists and that you have the required write permissions.

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables for configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory. Example content is as follows:

```hcl
# Log analysis instance configuration
log_analysis_instance_id = "your_rds_instance_id"

# Slow log export task configuration
slow_log_bucket_name = "your_obs_bucket"
slow_log_start_time  = "2026-08-12T00:00:00Z"
slow_log_end_time    = "2026-08-13T23:59:59Z"

# Binlog parse task configuration
binlog_binlog_type = "mysql"
binlog_file_name   = "mysql binlog file name"

# Binlog parse result export configuration
binlog_export_bucket_name         = "your_obs_bucket"
binlog_filter_start_time          = "2000-06-01T00:00:00+08:00"
binlog_filter_end_time            = "2099-06-02T00:00:00+08:00"
binlog_filter_types               = ["insert", "update", "delete", "ddl"]
binlog_filter_parse_double_insert = true
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; for other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="log_analysis_instance_id=your_rds_instance_id" -var="slow_log_bucket_name=your_obs_bucket"`
2. Environment variables: `export TF_VAR_log_analysis_instance_id=your_rds_instance_id`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the log analysis configurations
4. Run `terraform show` to view the created log analysis configurations

## Reference Information

- [Huawei Cloud DAS Product Documentation](https://support.huaweicloud.com/das/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DAS Log Analysis Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/das/log-analysis)
