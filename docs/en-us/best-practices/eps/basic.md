# Deploy Enterprise Project

## Application Scenario

Enterprise Project Management Service (EPS) is an enterprise-level resource management service provided by Huawei Cloud, enabling enterprises to plan, classify, and authorize cloud resources by project. With enterprise projects, enterprises can group resources of different businesses, departments, or environments into independent projects and implement fine-grained permission control together with Identity and Access Management (IAM).

This best practice will introduce how to use Terraform to automatically deploy an enterprise project, including the creation of the enterprise project and the configuration of its core parameters such as name, description, type, and enable status, helping you quickly get started with automated enterprise project management.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Enterprise Project (huaweicloud_enterprise_project)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/enterprise_project)

### Resource/Data Source Dependencies

```
huaweicloud_enterprise_project
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create an Enterprise Project

Add the following script in the TF file (such as main.tf) to create an enterprise project:

```hcl
# Create an enterprise project resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "enterprise_project_name" {
  description = "The name of the enterprise project"
  type        = string
}

variable "enterprise_project_description" {
  description = "The description of the enterprise project"
  type        = string
  default     = ""
}

variable "enterprise_project_type" {
  description = "The type of the enterprise project. Valid values are poc and prod"
  type        = string
  default     = "prod"
}

variable "enterprise_project_enable" {
  description = "Whether to enable the enterprise project"
  type        = bool
  default     = true
}

variable "skip_disable_on_destroy" {
  description = "Whether to skip disabling the enterprise project on destroy"
  type        = bool
  default     = false
}

variable "delete_flag" {
  description = "Whether to delete the enterprise project on destroy"
  type        = bool
  default     = true
}

resource "huaweicloud_enterprise_project" "test" {
  name                    = var.enterprise_project_name
  description             = var.enterprise_project_description
  type                    = var.enterprise_project_type
  enable                  = var.enterprise_project_enable
  skip_disable_on_destroy = var.skip_disable_on_destroy
  delete_flag             = var.delete_flag
}
```

**Parameter Description**:

- **name**: The name of the enterprise project, assigned by referencing the input variable enterprise_project_name. The name must be unique in the account and cannot contain any form of the word "default"
- **description**: The description of the enterprise project, assigned by referencing the input variable enterprise_project_description
- **type**: The type of the enterprise project, assigned by referencing the input variable enterprise_project_type. Valid values are poc and prod
- **enable**: Whether to enable the enterprise project, assigned by referencing the input variable enterprise_project_enable
- **skip_disable_on_destroy**: Whether to skip disabling the enterprise project on destroy, assigned by referencing the input variable skip_disable_on_destroy
- **delete_flag**: Whether to delete the enterprise project on destroy, assigned by referencing the input variable delete_flag. Ensure the project has no associated resources before enabling this option

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Enterprise project configuration
enterprise_project_name        = "tf-test-eps"
enterprise_project_description = "Terraform EPS basic example"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="enterprise_project_name=my-eps"`
2. Environment variables: `export TF_VAR_enterprise_project_name=my-eps`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the enterprise project
4. Run `terraform show` to view the created enterprise project

## Reference Information

- [Huawei Cloud Enterprise Project Management Service Product Documentation](https://support.huaweicloud.com/eps/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EPS Enterprise Project](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eps/basic)
