# Deploy Enterprise Project Supported Services Query

## Application Scenario

Enterprise Project Management Service (EPS) is an enterprise-level resource management service provided by Huawei Cloud, allowing enterprises to group, authorize, and perform cost accounting on cloud resources by project. Before using enterprise projects to aggregate and authorize resources, you usually need to know which cloud services and resource types are supported by EPS in the current region, so that you can plan the ownership relationship between enterprise projects and resources.

This best practice will introduce how to use Terraform to query the services supported by EPS, filtering by display language and service name through a data source, helping you quickly obtain the list of services available for enterprise project management and laying the foundation for subsequent resource grouping, authorization, and cost accounting.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Enterprise Project Supported Services (data.huaweicloud_enterprise_project_services)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/enterprise_project_services)

### Resource/Data Source Dependencies

```
data.huaweicloud_enterprise_project_services
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query the Services Supported by Enterprise Projects

Add the following script in the TF file (such as main.tf):

```hcl
# Query the services supported by enterprise projects in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "exact_locale" {
  description = "The display language for service and resource type names"
  type        = string
  default     = ""
  nullable    = false
}

variable "fuzzy_service_name" {
  description = "The fuzzy service name to filter"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_enterprise_project_services" "test" {
  locale  = var.exact_locale != "" ? var.exact_locale : null
  service = var.fuzzy_service_name != "" ? var.fuzzy_service_name : null
}
```

**Parameter Description**:

- **locale**: The display language for service and resource type names, assigned by referencing the input variable exact_locale; this parameter does not take effect when the variable is an empty string
- **service**: The fuzzy service name to filter, assigned by referencing the input variable fuzzy_service_name; this parameter does not take effect when the variable is an empty string

### 3. Filter the Query Result by Exact Service Name

Add the following script in the TF file (such as main.tf):

```hcl
# Filter the query result by exact service name in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "exact_service_name" {
  description = "The exact service name to filter"
  type        = string
  default     = ""
  nullable    = false
}

locals {
  matched_services = data.huaweicloud_enterprise_project_services.test.services

  # Filter the services supported by EPS by the user-specified exact service name
  matched_service_by_only_exact_service_name = var.exact_service_name != "" ? [
    for v in data.huaweicloud_enterprise_project_services.test.services : v if v.service == var.exact_service_name
  ] : data.huaweicloud_enterprise_project_services.test.services
}
```

**Parameter Description**:

- **matched_services**: All services returned by the data source, assigned by referencing the services attribute of the data source huaweicloud_enterprise_project_services.test
- **matched_service_by_only_exact_service_name**: The service list filtered by the exact service name, assigned by referencing the input variable exact_service_name; all services are returned when the variable is an empty string

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "<YOUR_ACCESS_KEY>"
secret_key  = "<YOUR_SECRET_KEY>"

# Filter variables
exact_locale       = "en-us"
fuzzy_service_name = "er"
exact_service_name = "ServiceStage"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="fuzzy_service_name=er"`
2. Environment variables: `export TF_VAR_fuzzy_service_name=er`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to query the services supported by enterprise projects:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start querying the services supported by enterprise projects
4. Run `terraform show` to view the queried services supported by enterprise projects

## Reference Information

- [Huawei Cloud Enterprise Project Management Service Product Documentation](https://support.huaweicloud.com/eps/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EPS Supported Services Query](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eps/query-supported-services)
