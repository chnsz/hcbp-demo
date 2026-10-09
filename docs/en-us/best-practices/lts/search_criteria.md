# Deploy Search Criteria

## Application Scenario

Log Tank Service (LTS) is a one-stop log management service provided by Huawei Cloud, supporting log collection, storage, query, and analysis. In daily operations, users often need to run the same query statements repeatedly against a log stream, for example filtering raw logs by keyword or aggregating metrics with visualization statements. Search criteria allow you to save these frequently used query statements so that they can be reused quickly in the console without rewriting the queries.

This best practice will introduce how to use Terraform to automatically deploy LTS search criteria, including the creation of a log group, a log stream, and a search criteria, supporting both the ORIGINALLOG and VISUALIZATION criteria types, and managing log resources uniformly through tags and log expiration days.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Log Group (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [Log Stream (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [Search Criteria (huaweicloud_lts_search_criteria)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_search_criteria)

### Resource/Data Source Dependencies

```
huaweicloud_lts_group
    └── huaweicloud_lts_stream
            └── huaweicloud_lts_search_criteria
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Log Group

Add the following script to the TF file (such as main.tf) to create a log group:

```hcl
# Create a log group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  description = "The enterprise project ID of the log group and log stream"
  type        = string
  default     = null
}

resource "huaweicloud_lts_group" "test" {
  group_name            = var.group_name
  ttl_in_days           = var.group_log_expiration_days
  tags                  = var.group_tags
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **group_name**: The name of the log group, assigned by referencing the input variable group_name
- **ttl_in_days**: The log expiration days of the log group, assigned by referencing the input variable group_log_expiration_days, defaulting to 14 days
- **tags**: The tags of the log group, assigned by referencing the input variable group_tags, used for resource categorization and cost allocation
- **enterprise_project_id**: The enterprise project ID of the log group, assigned by referencing the input variable enterprise_project_id, used for enterprise project isolation

### 3. Create a Log Stream

Add the following script to the TF file (such as main.tf) to create a log stream:

```hcl
# Create a log stream resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **group_id**: The ID of the log group to which the log stream belongs, assigned by referencing the log group huaweicloud_lts_group.test.id created in the previous step
- **stream_name**: The name of the log stream, assigned by referencing the input variable stream_name
- **ttl_in_days**: The log expiration days of the log stream, assigned by referencing the input variable stream_log_expiration_days; a value of null or -1 indicates that it is consistent with the log group
- **tags**: The tags of the log stream, assigned by referencing the input variable stream_tags
- **enterprise_project_id**: The enterprise project ID of the log stream, assigned by referencing the input variable enterprise_project_id
- **is_favorite**: Whether to favorite the log stream, assigned by referencing the input variable stream_is_favorite

### 4. Create a Search Criteria

Add the following script to the TF file (such as main.tf) to create a search criteria:

```hcl
# Create a search criteria resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "search_criteria" {
  description = "The content of the search criteria"
  type        = string
}

variable "search_criteria_name" {
  description = "The name of the search criteria"
  type        = string
}

variable "search_criteria_type" {
  description = "The type of the search criteria. Available types are ORIGINALLOG and VISUALIZATION"
  type        = string
  default     = "ORIGINALLOG"
}

resource "huaweicloud_lts_search_criteria" "test" {
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id

  criteria = var.search_criteria
  name     = var.search_criteria_name
  type     = var.search_criteria_type
}
```

**Parameter Description**:
- **log_group_id**: The ID of the log group to which the search criteria belongs, assigned by referencing the log group huaweicloud_lts_group.test.id created above
- **log_stream_id**: The ID of the log stream to which the search criteria belongs, assigned by referencing the log stream huaweicloud_lts_stream.test.id created above
- **criteria**: The content of the search criteria, assigned by referencing the input variable search_criteria
- **name**: The name of the search criteria, assigned by referencing the input variable search_criteria_name; it can only contain English letters, numbers, Chinese characters, hyphens, underscores, and periods, and cannot start with a period or underscore or end with a period
- **type**: The type of the search criteria, assigned by referencing the input variable search_criteria_type; available values are ORIGINALLOG (raw logs) and VISUALIZATION (visualized logs)

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# LTS resource variables
group_name           = "tf_test_log_group"
stream_name          = "tf_test_log_stream"
search_criteria      = "content:test"
search_criteria_name = "tf_test_search_criteria"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="group_name=my-group"`
2. Environment variables: `export TF_VAR_group_name=my-group`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the search criteria
4. Run `terraform show` to view the created search criteria

## Reference Information

- [Huawei Cloud Log Tank Service Product Documentation](https://support.huaweicloud.com/lts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For LTS Search Criteria](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/lts/search-criteria)
