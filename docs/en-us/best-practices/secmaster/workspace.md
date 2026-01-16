# Deploy Workspace

## Application Scenario

Security Master (SecMaster) is a security situation awareness and security operations platform provided by Huawei Cloud, supporting unified management, analysis, and response of security events, helping you achieve automation and intelligence in security operations. Workspace is a fundamental resource of SecMaster, used to isolate and manage security resources for different business scenarios. By creating workspaces, independent security operations environments can be created under specified projects, achieving unified management and isolation of security resources. This best practice introduces how to use Terraform to automatically deploy workspaces, including workspace basic information, project configuration, enterprise project configuration, and tag configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Workspace Resource (huaweicloud_secmaster_workspace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_workspace)

### Resource/Data Source Dependencies

```text
huaweicloud_secmaster_workspace
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create Workspace

Add the following script to the TF file (such as main.tf) to create a workspace:

```hcl
variable "workspace_name" {
  description = "The name of the workspace"
  type        = string
}

variable "workspace_project_name" {
  description = "The name of the project to in which to create the workspace"
  type        = string
}

variable "workspace_description" {
  description = "The description of the workspace"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the workspace belongs"
  type        = string
  default     = null
}

variable "workspace_tags" {
  description = "The key/value pairs to associate with the workspace"
  type        = map(string)
  default     = {}
}

# Create workspace resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_secmaster_workspace" "test" {
  name                  = var.workspace_name
  project_name          = var.workspace_project_name
  description           = var.workspace_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.workspace_tags
}
```

**Parameter Description**:
- **name**: Workspace name, assigned by referencing the input variable `workspace_name`
- **project_name**: Project name, assigned by referencing the input variable `workspace_project_name`, used to specify the project in which to create the workspace
- **description**: Workspace description, assigned by referencing the input variable `workspace_description`, optional parameter
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable `enterprise_project_id`, used to specify the enterprise project to which the workspace belongs, optional parameter
- **tags**: Tags, assigned by referencing the input variable `workspace_tags`, used to add key-value pair tags to the workspace, optional parameter

> Note: Workspaces must be created under specified projects. Enterprise project IDs are used to achieve unified management and isolation of resources. If not specified, the default enterprise project is used. Tags can be used for resource classification and management.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Workspace basic information (Required)
workspace_name         = "tf_test_workspace"
workspace_project_name = "cn-north-4"
workspace_description  = "This is a SecMaster workspace created by Terraform"

# Enterprise project configuration (Optional)
enterprise_project_id = "0"

# Tag configuration (Optional)
workspace_tags = {
  owner = "terraform"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="workspace_name=tf_test_workspace" -var="workspace_project_name=cn-north-4"`
2. Environment variables: `export TF_VAR_workspace_name=tf_test_workspace`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the workspace
4. Run `terraform show` to view the created workspace

## Reference Information

- [Huawei Cloud SecMaster Product Documentation](https://support.huaweicloud.com/secmaster/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Workspace](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/workspace)
