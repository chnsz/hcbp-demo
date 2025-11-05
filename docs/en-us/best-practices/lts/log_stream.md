# Deploy Log Stream

## Application Scenario

Huawei Cloud Log Tank Service (LTS) log stream functionality is a fundamental component of log management, used to organize and store log data. By creating log groups and log streams, you can implement log classification management, lifecycle control, tag classification, and other functions. Log streams support multiple log collection methods, can receive log data from different applications and services, and provide powerful query and analysis capabilities.

This best practice is particularly suitable for scenarios that require centralized management of application logs, implementing log classification storage, and building log monitoring and analysis systems, such as application operation monitoring, security auditing, business data analysis, etc. This best practice will introduce how to use Terraform to automatically deploy LTS log streams, including log group and log stream creation, implementing complete log management infrastructure.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Log Group Resource (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [Log Stream Resource (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)

### Resource/Data Source Dependencies

```
huaweicloud_lts_group
    └── huaweicloud_lts_stream
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Log Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a log group resource:

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

variable "group_tags" {
  description = "The tags of the log group"
  type        = map(string)
  default     = {}
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the log group"
  type        = string
  default     = null
}

# Create a log group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_lts_group" "test" {
  group_name            = var.group_name
  ttl_in_days           = var.group_log_expiration_days
  tags                  = var.group_tags
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **group_name**: Log group name, assigned by referencing the input variable group_name
- **ttl_in_days**: Log expiration days for the log group, assigned by referencing the input variable group_log_expiration_days, default value is 14
- **tags**: Log group tags, assigned by referencing the input variable group_tags, default value is empty map
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is null

### 3. Create Log Stream

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

variable "stream_tags" {
  description = "The tags of the log stream"
  type        = map(string)
  default     = {}
}

variable "stream_is_favorite" {
  description = "Whether to favorite the log stream"
  type        = bool
  default     = false
}

# Create a log stream resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_lts_stream" "test" {
  group_id              = huaweicloud_lts_group.test.id
  stream_name           = var.stream_name
  ttl_in_days           = var.stream_log_expiration_days
  tags                  = var.stream_tags
  enterprise_project_id = var.enterprise_project_id
  is_favorite           = var.stream_is_favorite
}
```

**Parameter Description**:
- **group_id**: Log group ID that the log stream belongs to, referencing the ID of the previously created log group resource
- **stream_name**: Log stream name, assigned by referencing the input variable stream_name
- **ttl_in_days**: Log expiration days for the log stream, assigned by referencing the input variable stream_log_expiration_days, default value is null (inherits log group settings)
- **tags**: Log stream tags, assigned by referencing the input variable stream_tags, default value is empty map
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is null
- **is_favorite**: Whether to favorite the log stream, assigned by referencing the input variable stream_is_favorite, default value is false

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Log group configuration
group_name = "tf_test_server_group"

# Log stream configuration
stream_name = "tf_test_stream"
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

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating log streams
4. Run `terraform show` to view the created log stream details

## Reference Information

- [Huawei Cloud Log Tank Service Product Documentation](https://support.huaweicloud.com/lts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference LTS Log Stream](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/lts/log-stream)
