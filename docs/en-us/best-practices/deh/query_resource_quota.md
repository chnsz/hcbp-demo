# Deploy Query Resource Quota

## Application Scenario

Dedicated Host (DEH) resource quota query is a quota query function provided by the DEH service, used to query resource quota information related to dedicated hosts. By querying resource quotas, you can understand the quota usage of dedicated host resources under the current account, including used quotas, available quotas, and exhausted quotas, helping you reasonably plan resource usage and avoid resource creation failures due to insufficient quotas. Automating DEH resource quota queries through Terraform can ensure standardized and consistent quota queries, improving operational efficiency. This best practice will introduce how to use Terraform to automatically query DEH resource quotas.

## Related Resources/Data Sources

This best practice involves the following main data source:

### Data Sources

- [DEH Quotas Data Source (huaweicloud_deh_quotas)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/deh_quotas)

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query DEH Quotas Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query DEH quota information:

```hcl
variable "host_type" {
  description = "The type of the dedicated host to filter quotas"
  type        = string
  default     = ""
}

# Query DEH quotas data source in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
data "huaweicloud_deh_quotas" "test" {
  resource = var.host_type != "" ? var.host_type : null
}

# Query used quotas
locals {
  quotas_with_usage = [for v in data.huaweicloud_deh_quotas.test.quota_set : v if v.used > 0]

  # Query available quotas
  quotas_available = [for v in data.huaweicloud_deh_quotas.test.quota_set : v if v.hard_limit > v.used]

  # Query exhausted quotas
  quotas_exhausted = [for v in data.huaweicloud_deh_quotas.test.quota_set : v if v.hard_limit == v.used]
}
```

**Parameter Description**:
- **resource**: Dedicated host type, assigned by referencing the input variable host_type, optional parameter, default value is null (query all quotas)
- **quota_set**: Quota set, contains the following fields:
  - **type**: Quota type
  - **used**: Number of used quotas
  - **hard_limit**: Quota limit
  - **unit**: Quota unit

**Local Values Description**:
- **quotas_with_usage**: List of used quotas, used to identify which resources are in use
- **quotas_available**: List of available quotas, used to identify which resources can still be created
- **quotas_exhausted**: List of exhausted quotas, used to identify which resources need quota increase

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Dedicated Host Type Configuration (Optional, used to filter quotas for specific host types)
host_type = "s6"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
   - If `host_type` is empty string or not set, quotas for all host types will be queried
   - If `host_type` is set to a specific value (such as "s6"), only quotas for that host type will be queried
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="host_type=s6"`
2. Environment variables: `export TF_VAR_host_type=s6`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to query DEH resource quotas:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the data source query plan
3. After confirming that the query plan is correct, run `terraform apply` to start querying quota information
4. Run `terraform output` to view the quota query results

**Output Description**:

After the query is completed, you can view quota information in the following ways:

- View used quotas: `terraform output quotas_with_usage`
- View available quotas: `terraform output quotas_available`
- View exhausted quotas: `terraform output quotas_exhausted`

> Note: Quota queries can help you understand the quota usage of dedicated host resources under the current account. By querying used quotas, you can identify which resources are in use; by querying available quotas, you can identify which resources can still be created; by querying exhausted quotas, you can identify which resources need quota increase. If the host_type parameter is empty, quota information for all host types will be queried.

## Reference Information

- [Huawei Cloud DEH Product Documentation](https://support.huaweicloud.com/deh/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Query Resource Quota](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/deh/query-resource-quota)
