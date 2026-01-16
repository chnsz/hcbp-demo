# Deploy Template

## Application Scenario

Resource Governance Center (RGC) is a resource governance service provided by Huawei Cloud, supporting multi-account management, organizational unit management, blueprint configuration, and other functions to help you uniformly manage and govern cloud resources. By creating RGC templates, deployment blueprints for resources can be defined, achieving automated resource deployment and management. Templates can be predefined templates or customized templates. Predefined templates are provided by Huawei Cloud, while customized templates can be configured according to actual needs. This best practice introduces how to use Terraform to automatically deploy RGC templates, including the creation of predefined templates and customized templates.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Template Resource (huaweicloud_rgc_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rgc_template)

### Resource/Data Source Dependencies

```text
huaweicloud_rgc_template
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create Template

Add the following script to the TF file (such as main.tf) to create an RGC template:

```hcl
variable "template_name" {
  description = "The name of the template"
  type        = string
}

variable "template_type" {
  description = "The type of the template"
  type        = string
  default     = "predefined"
}

variable "template_description" {
  description = "The description of the customized template"
  type        = string
  default     = null
}

variable "template_body" {
  description = "The content of the customized template"
  type        = string
  default     = null
}

# Create template resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_rgc_template" "test" {
  template_name        = var.template_name
  template_type        = var.template_type
  template_description = var.template_description
  template_body        = var.template_body
}
```

**Parameter Description**:
- **template_name**: Template name, assigned by referencing the input variable `template_name`
- **template_type**: Template type, assigned by referencing the input variable `template_type`. Optional values include `predefined` (predefined template) and `customized` (customized template), default is `predefined`
- **template_description**: Description of the customized template, assigned by referencing the input variable `template_description`. Only valid when `template_type` is `customized`, optional parameter
- **template_body**: Content of the customized template, assigned by referencing the input variable `template_body`. Only valid when `template_type` is `customized`, optional parameter

> Note: Predefined templates are provided by Huawei Cloud. When creating, only the template name and type need to be specified. Customized templates require template description and template content. Template content is usually blueprint configuration in JSON format.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Template basic information (Required)
template_name = "tf_test_template"
template_type = "predefined"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

If creating a customized template, you can add the following content to the `terraform.tfvars` file:

```hcl
# Customized template configuration
template_name        = "tf_test_customized_template"
template_type        = "customized"
template_description = "This is a customized template for resource deployment"
template_body        = jsonencode({
  version = "1.0"
  resources = {
    # Template content configuration
  }
})
```

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="template_name=tf_test_template" -var="template_type=predefined"`
2. Environment variables: `export TF_VAR_template_name=tf_test_template`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the template
4. Run `terraform show` to view the created template

## Reference Information

- [Huawei Cloud RGC Product Documentation](https://support.huaweicloud.com/rgc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Template](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rgc/template)
