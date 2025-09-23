# Deploy Compliance Package

## Application Scenario

Configuration Audit (Config) is a one-stop compliance management service provided by Huawei Cloud, helping users continuously monitor and evaluate the configuration compliance of cloud resources. Config service provides pre-built compliance rule packages and custom rules, supporting multiple compliance frameworks and standards, helping enterprises establish a comprehensive compliance management system.

Compliance packages are a core function of Config service, used to define and manage a set of related compliance rules. Through compliance packages, enterprises can quickly deploy rule sets that comply with specific compliance frameworks or standards, achieving automated compliance checks and evaluations. Compliance packages support pre-built templates and custom configurations, providing flexible rule management capabilities and comprehensive compliance management solutions for enterprises. This best practice will introduce how to use Terraform to automatically deploy Config compliance packages, including rule package template queries and compliance package creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [RMS Assignment Package Templates Query Data Source (data.huaweicloud_rms_assignment_package_templates)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rms_assignment_package_templates)

### Resources

- [RMS Assignment Package Resource (huaweicloud_rms_assignment_package)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rms_assignment_package)

### Resource/Data Source Dependencies

```
data.huaweicloud_rms_assignment_package_templates.test
    └── huaweicloud_rms_assignment_package.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Rule Package Templates via Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the result of which will be used to create the compliance package:

```hcl
variable "template_key" {
  description = "Built-in rule package template name"
  type        = string
  default     = ""
}

# Get all rule package template information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create the compliance package
data "huaweicloud_rms_assignment_package_templates" "test" {
  template_key = var.template_key
}
```

**Parameter Description**:
- **template_key**: Built-in rule package template name, assigned by referencing the input variable template_key

### 3. Create Compliance Package

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a compliance package resource:

```hcl
variable "assignment_package_name" {
  description = "Rule package name"
  type        = string
}

# Create a compliance package resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_rms_assignment_package" "test" {
  name         = var.assignment_package_name
  template_key = data.huaweicloud_rms_assignment_package_templates.test.templates.0.template_key

  dynamic "vars_structure" {
    for_each = data.huaweicloud_rms_assignment_package_templates.test.templates.0.parameters

    content {
      var_key   = vars_structure.value["name"]
      var_value = vars_structure.value["default_value"]
    }
  }
}
```

**Parameter Description**:
- **name**: Rule package name, assigned by referencing the input variable assignment_package_name
- **template_key**: Template key name, assigned by referencing the first template key name from the rule package template data source
- **vars_structure**: Variable structure, dynamically creates rule package parameter configuration
  - **var_key**: Variable key name, uses the template parameter name
  - **var_value**: Variable value, uses the template parameter default value

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Rule package template configuration
template_key = "Operational-Best-Practices-for-ECS.tf.json"

# Compliance package configuration
assignment_package_name = "tf_test_assignment_package"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="assignment_package_name=my-package" -var="template_key=my-template"`
2. Environment variables: `export TF_VAR_assignment_package_name=my-package`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the compliance package
4. Run `terraform show` to view the details of the created compliance package

## Reference Information

- [Huawei Cloud Config Product Documentation](https://support.huaweicloud.com/rms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Config Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rms)
