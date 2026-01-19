# Deploy Query Resource Types

## Application Scenario

Tag Management Service (TMS) is a tag management service provided by Huawei Cloud, supporting adding, modifying, and deleting tags for cloud resources, helping you achieve resource classification management and cost analysis. Resource type query is an important function of TMS service, used to query cloud resource types that support tag management. By querying resource types, service names and resource type lists that support tag management can be obtained, providing basic information for subsequent tag management operations. This best practice introduces how to use Terraform to automatically deploy resource type queries, including exact matching and fuzzy matching of service names, fuzzy matching of resource type names, and other query methods.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Resource Types Query Data Source (data.huaweicloud_tms_resource_types)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/tms_resource_types)

### Resource/Data Source Dependencies

```text
data.huaweicloud_tms_resource_types
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Query Resource Types

Add the following script to the TF file (such as main.tf) to query resource types:

```hcl
variable "exact_service_name" {
  description = "The exact service name to filter"
  type        = string
  default     = ""
}

variable "fuzzy_service_name" {
  description = "The fuzzy service name to filter"
  type        = string
  default     = ""
}

variable "fuzzy_resource_type_name" {
  description = "The fuzzy resource type name to filter"
  type        = string
  default     = ""
}

# Query resource types data source
data "huaweicloud_tms_resource_types" "test" {
  service_name = var.exact_service_name != "" ? var.exact_service_name : null
}

# Perform fuzzy matching and filtering through local values
locals {
  # All service names registered on TMS service (using fuzzy matching based on user-specified service name)
  regex_matched_service_names = distinct(var.fuzzy_service_name != "" ? [
    for v in data.huaweicloud_tms_resource_types.test.types[*].service_name : v if length(regexall(var.fuzzy_service_name, v)) > 0
  ] : data.huaweicloud_tms_resource_types.test.types[*].service_name)

  # All resource types (object including the resource type name and the service name to which the resource type
  # belongs) registered on TMS service (using fuzzy matching based on user-specified service name)
  regex_matched_resource_types_by_only_fuzzy_service_name = var.fuzzy_service_name != "" ? [
    for v in data.huaweicloud_tms_resource_types.test.types : v if length(regexall(var.fuzzy_service_name, v.service_name)) > 0
  ] : data.huaweicloud_tms_resource_types.test.types

  # All resource types (object including the resource type name and the service name to which the resource type
  # belongs) registered on TMS service (using fuzzy matching based on user-specified service name or resource type name)
  regex_matched_resource_types = var.fuzzy_resource_type_name != "" ? [
    for v in local.regex_matched_resource_types_by_only_fuzzy_service_name : v if length(regexall(var.fuzzy_resource_type_name, v.name)) > 0
  ] : local.regex_matched_resource_types_by_only_fuzzy_service_name
}
```

**Parameter Description**:
- **service_name**: Service name, assigned by referencing the input variable `exact_service_name`, used for exact matching of service names, optional parameter
- **regex_matched_service_names**: Service name list through fuzzy matching using local values `locals`. When `fuzzy_service_name` is not empty, fuzzy matching is performed; otherwise, all service names are returned
- **regex_matched_resource_types_by_only_fuzzy_service_name**: Resource type list through fuzzy matching based on service names using local values `locals`. When `fuzzy_service_name` is not empty, fuzzy matching is performed; otherwise, all resource types are returned
- **regex_matched_resource_types**: Resource type list through fuzzy matching based on service names or resource type names using local values `locals`. When `fuzzy_resource_type_name` is not empty, fuzzy matching is performed; otherwise, results based on service name matching are returned

> Note: Resource type query supports both exact matching and fuzzy matching. Exact matching specifies service names through the `service_name` parameter, and fuzzy matching uses regular expressions through local values. Query results include service names and resource type names, which can be used for subsequent tag management operations.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Fuzzy match service name (Optional)
fuzzy_service_name = "ccm"

# Fuzzy match resource type name (Optional)
fuzzy_resource_type_name = "certificate"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="fuzzy_service_name=ccm" -var="fuzzy_resource_type_name=certificate"`
2. Environment variables: `export TF_VAR_fuzzy_service_name=ccm`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to query resource types:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the query plan
3. Run `terraform apply` to execute the query operation
4. Run `terraform show` to view query results, or use `terraform output` to output local value results

## Reference Information

- [Huawei Cloud TMS Product Documentation](https://support.huaweicloud.com/tms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Query Resource Types](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/tms/query-resource-types)
