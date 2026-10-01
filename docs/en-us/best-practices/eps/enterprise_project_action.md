# Deploy Enterprise Project Enable/Disable Action

## Application Scenario

Enterprise Project Management Service (EPS) is an enterprise-level resource management service provided by Huawei Cloud, allowing enterprises to group, authorize, and perform cost accounting on cloud resources by project. During daily operations, an enterprise project may need to be temporarily enabled or disabled based on business cycles or compliance requirements, for example, disabling an enterprise project during business downtime to restrict resource operations, or re-enabling it after business recovery.

This best practice will introduce how to use Terraform to automatically perform enable or disable actions on an enterprise project. This practice supports two usage modes: first creating a new enterprise project and then performing the enable/disable action on it, or directly performing the enable/disable action on an existing enterprise project, thereby meeting enterprise project state management requirements in different scenarios.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Enterprise Project (huaweicloud_enterprise_project)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/enterprise_project)
- [Enterprise Project Action (huaweicloud_enterprise_project_action)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/enterprise_project_action)

### Resource/Data Source Dependencies

```
huaweicloud_enterprise_project
    └── huaweicloud_enterprise_project_action
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create an Enterprise Project (Optional)

Add the following script to the TF file (such as main.tf) to create an enterprise project. When no existing enterprise project ID is provided, a new enterprise project will be created:

```hcl
# Create an enterprise project resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
}

variable "enterprise_project_name" {
  description = "The name of the enterprise project"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.enterprise_project_id != "" || var.enterprise_project_name != ""
    error_message = "enterprise_project_name must be provided if enterprise_project_id is not provided."
  }
}

variable "enterprise_project_description" {
  description = "The description of the enterprise project"
  type        = string
  default     = ""
}

variable "enterprise_project_type" {
  description = "The type of the enterprise project"
  type        = string
  default     = "prod"
}

variable "enterprise_project_enable" {
  description = "Whether to enable the enterprise project"
  type        = bool
  default     = true
}

variable "delete_flag" {
  description = "Whether to delete the enterprise project on destroy"
  type        = bool
  default     = true
}

resource "huaweicloud_enterprise_project" "test" {
  count = var.enterprise_project_id == "" ? 1 : 0

  name        = var.enterprise_project_name
  description = var.enterprise_project_description
  type        = var.enterprise_project_type
  enable      = var.enterprise_project_enable
  delete_flag = var.delete_flag

  lifecycle {
    ignore_changes = [
      enable
    ]
  }
}
```

**Parameter Description**:
- **count**: Controls whether to create the enterprise project through a conditional expression; it is created when the input variable enterprise_project_id is an empty string, otherwise it is not created
- **name**: Assigned by referencing the input variable enterprise_project_name, representing the name of the enterprise project
- **description**: Assigned by referencing the input variable enterprise_project_description, representing the description of the enterprise project
- **type**: Assigned by referencing the input variable enterprise_project_type, representing the type of the enterprise project, defaulting to prod
- **enable**: Assigned by referencing the input variable enterprise_project_enable, indicating whether to enable the enterprise project, defaulting to true
- **delete_flag**: Assigned by referencing the input variable delete_flag, indicating whether to delete the enterprise project on destroy, defaulting to true
- **lifecycle.ignore_changes**: Ignores changes to the enable attribute to avoid conflicts with the subsequent action resource

### 3. Perform the Enterprise Project Enable/Disable Action

Add the following script to the TF file to perform the enable or disable action on the enterprise project:

```hcl
# Perform the enterprise project action in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "enterprise_project_action" {
  description = "The action to perform on the enterprise project. Valid values are enable and disable"
  type        = string
  default     = "disable"
}

resource "huaweicloud_enterprise_project_action" "test" {
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : try(huaweicloud_enterprise_project.test[0].id, "")
  action                = var.enterprise_project_action
}
```

**Parameter Description**:
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; if not provided, it references the ID of the enterprise project created in the previous step
- **action**: Assigned by referencing the input variable enterprise_project_action, representing the action to perform, with valid values enable and disable, defaulting to disable

> Note: `huaweicloud_enterprise_project_action` is a one-time action resource. Deleting it only removes the record from the Terraform state and does not revert the action. In addition, the **poc** type enterprise project does not support the disable operation.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Create a new enterprise project and perform the action
enterprise_project_name = "tf-test-eps"

# Or perform the action on an existing enterprise project
# enterprise_project_id = "your-enterprise-project-id"
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

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to perform the enterprise project enable/disable action
4. Run `terraform show` to view the executed enterprise project action

## Reference Information

- [Huawei Cloud Enterprise Project Management Service Product Documentation](https://support.huaweicloud.com/eps/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EPS Enterprise Project Action](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eps/enterprise-project-action)
