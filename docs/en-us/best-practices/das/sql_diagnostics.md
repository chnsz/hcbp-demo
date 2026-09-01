# Deploy SQL Diagnostics

## Application Scenario

Data Admin Service (DAS) provides SQL diagnostics and intelligent O&M capabilities. It supports batch-enabling SQL collection switches for database instances and enabling the search path switch for database connections, helping DBAs analyze slow SQL, full SQL, and other runtime data to improve troubleshooting efficiency. This best practice introduces how to use Terraform to automatically deploy DAS SQL diagnostics configurations, including batch-setting the SQL switch and enabling the search path switch for a database connection.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Batch Set SQL Switch Resource (huaweicloud_das_batch_set_sql_switch)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_batch_set_sql_switch)
- [Search Path Switch Resource (huaweicloud_das_search_path_switch)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_search_path_switch)

### Resource/Data Source Dependencies

```text
huaweicloud_das_batch_set_sql_switch

huaweicloud_das_search_path_switch
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the configuration introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Batch Set SQL Switch

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a batch set SQL switch resource:

```hcl
variable "sql_diagnostics_engine_type" {
  description = "The engine type of the instances"
  type        = string
}

variable "batch_sql_switch_on" {
  description = "Whether to enable the SQL switch"
  type        = bool
}

variable "batch_sql_switch_type" {
  description = "The type of SQL switch to set"
  type        = string
}

variable "batch_sql_instance_ids" {
  description = "The list of instance IDs"
  type        = list(string)
}

variable "batch_sql_retention_hours" {
  description = "The retention hours of the SQL data"
  type        = number
  default     = null
}

# Create a batch set SQL switch resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_batch_set_sql_switch" "test" {
  engine_type     = var.sql_diagnostics_engine_type
  switch_on       = var.batch_sql_switch_on
  switch_type     = var.batch_sql_switch_type
  instance_ids    = var.batch_sql_instance_ids
  retention_hours = var.batch_sql_retention_hours
}
```

**Parameter Description**:
- **engine_type**: Engine type of the instances, assigned by referencing the input variable `sql_diagnostics_engine_type`
- **switch_on**: Whether to enable the SQL switch, assigned by referencing the input variable `batch_sql_switch_on`
- **switch_type**: Type of SQL switch to set, assigned by referencing the input variable `batch_sql_switch_type`
- **instance_ids**: List of instance IDs, assigned by referencing the input variable `batch_sql_instance_ids`
- **retention_hours**: Retention hours of the SQL data, assigned by referencing the input variable `batch_sql_retention_hours`

> Note: Ensure that the target database instances already exist before deployment. Configure the SQL data retention period reasonably based on business requirements.

### 3. Enable Search Path Switch

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a search path switch resource:

```hcl
variable "search_path_connection_id" {
  description = "The ID of the database connection (DB user ID)"
  type        = string
}

variable "search_path_switch_on" {
  description = "Whether to enable the search path switch"
  type        = bool
}

# Create a search path switch resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_das_search_path_switch" "test" {
  connection_id = var.search_path_connection_id
  switch_on     = var.search_path_switch_on
}
```

**Parameter Description**:
- **connection_id**: ID of the database connection (DB user ID), assigned by referencing the input variable `search_path_connection_id`
- **switch_on**: Whether to enable the search path switch, assigned by referencing the input variable `search_path_switch_on`

> Note: Ensure that the corresponding database connection (database user) already exists before deployment.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables for configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory. Example content is as follows:

```hcl
# Batch set SQL switch configuration
sql_diagnostics_engine_type = "MySQL"
batch_sql_switch_on         = true
batch_sql_switch_type       = "DAS_QUERY"
batch_sql_instance_ids      = ["tf_test_instance_id_1", "tf_test_instance_id_2"]
batch_sql_retention_hours   = 24

# Search path switch configuration
search_path_connection_id = "tf_test_connection_id"
search_path_switch_on     = true
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; for other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="batch_sql_switch_on=true" -var="search_path_switch_on=true"`
2. Environment variables: `export TF_VAR_sql_diagnostics_engine_type=MySQL`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the SQL diagnostics configurations
4. Run `terraform show` to view the created SQL diagnostics configurations

## Reference Information

- [Huawei Cloud DAS Product Documentation](https://support.huaweicloud.com/das/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DAS SQL Diagnostics Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/das/sql-diagnostics)
