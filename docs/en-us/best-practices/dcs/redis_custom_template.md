# Deploy Redis Custom Template

## Application Scenario

Distributed Cache Service (DCS) provides system preset templates and user-defined templates for quickly creating Redis instances that meet business requirements. When the parameter configuration of a system template cannot meet specific business requirements, you can create a custom template based on an existing template to override and adjust parameters, achieving reuse and standardization of parameter configuration.

This best practice will introduce how to use Terraform to create a DCS Redis custom template based on a system or user template and override some parameter configurations, helping you efficiently manage Redis parameter templates through Infrastructure as Code (IaC).

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DCS Custom Template (huaweicloud_dcs_custom_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_custom_template)

### Resource/Data Source Dependencies

```
huaweicloud_dcs_custom_template
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified working directory, and ensure that it (or other TF files in the same level directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create DCS Redis Custom Template

Add the following script to the TF file (such as main.tf):

```hcl
# Create DCS Redis custom template in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "source_template_id" {
  description = "The ID of the source template to create the custom template from"
  type        = string
}

variable "source_type" {
  description = "The type of the source template. Valid values: sys, user"
  type        = string
  default     = "sys"
}

variable "template_name" {
  description = "The name of the custom template"
  type        = string
}

variable "template_description" {
  description = "The description of the custom template"
  type        = string
  default     = ""
}

variable "template_params" {
  description = "The template params to override, mapping param names to values"
  type        = map(string)
  default     = {
    "timeout" = "200"
  }
}

resource "huaweicloud_dcs_custom_template" "test" {
  template_id = var.source_template_id
  source_type = var.source_type
  name        = var.template_name
  description = var.template_description

  dynamic "params" {
    for_each = var.template_params

    content {
      param_name  = params.key
      param_value = params.value
    }
  }
}
```

**Parameter Description**:
- **template_id**: Assigned by referencing the input variable source_template_id, specifying the ID of the source template.
- **source_type**: Assigned by referencing the input variable source_type, specifying the type of the source template, supporting sys (system template) and user (user template).
- **name**: Assigned by referencing the input variable template_name, specifying the name of the custom template.
- **description**: Assigned by referencing the input variable template_description, specifying the description of the custom template.
- **params**: Assigned by referencing the input variable template_params, overriding template parameters in key-value pairs, where param_name is the parameter name and param_value is the parameter value.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign values to configurations. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through `tfvars` files, avoiding repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Fill in according to script variables; use placeholders for sensitive information
source_template_id   = "16"
source_type          = "sys"
template_name        = "tf_test_custom_template"
template_description = "terraform test custom template"
template_params      = {
  "timeout"          = "200"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="template_name=my-template"`
2. Environment variables: `export TF_VAR_template_name=my-template`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DCS Redis custom template
4. Run `terraform show` to view the created DCS Redis custom template

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Custom Template](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-custom-template)
