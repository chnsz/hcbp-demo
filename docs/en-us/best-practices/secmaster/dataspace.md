# Deploy Dataspace

## Application Scenario

SecMaster is a next-generation cloud native security operations center provided by Huawei Cloud, offering capabilities such as cloud asset management, security posture management, security information and event management, and security orchestration and automatic response. A dataspace is a data storage and isolation unit under a SecMaster workspace, used to store and independently manage security data within the workspace.

This best practice will introduce how to use Terraform to automatically deploy a SecMaster dataspace, including the creation of a workspace and a dataspace under it, helping you quickly build the data storage foundation of SecMaster.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [SecMaster Workspace (huaweicloud_secmaster_workspace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_workspace)
- [SecMaster Dataspace (huaweicloud_secmaster_dataspace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_dataspace)

### Resource/Data Source Dependencies

```
huaweicloud_secmaster_workspace
    └── huaweicloud_secmaster_dataspace
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a SecMaster Workspace

Add the following script in the TF file (such as main.tf) to create a SecMaster workspace:

```hcl
# Create a SecMaster workspace resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "workspace_name" {
  description = "The name of the SecMaster workspace"
  type        = string
}

variable "workspace_description" {
  description = "The description of the SecMaster workspace"
  type        = string
  default     = "Created by Terraform"
}

resource "huaweicloud_secmaster_workspace" "test" {
  name         = var.workspace_name
  project_name = var.region_name
  description  = var.workspace_description
}
```

**Parameter Description**:
- **name**: The workspace name, assigned by referencing the input variable workspace_name
- **project_name**: The name of the project to which the workspace belongs, assigned by referencing the input variable region_name
- **description**: The workspace description, assigned by referencing the input variable workspace_description

### 3. Create a SecMaster Dataspace

Add the following script in the TF file (such as main.tf) to create a SecMaster dataspace:

```hcl
# Create a SecMaster dataspace resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dataspace_name" {
  description = "The name of the SecMaster dataspace. The name can only contain English letters, digits and hyphens (-), and cannot start or end with a hyphen (-), nor can they appear consecutively. Valid length: 5-63"
  type        = string
}

variable "dataspace_description" {
  description = "The description of the SecMaster dataspace"
  type        = string
  default     = "Created by Terraform"
}

resource "huaweicloud_secmaster_dataspace" "test" {
  workspace_id   = huaweicloud_secmaster_workspace.test.id
  dataspace_name = var.dataspace_name
  description    = var.dataspace_description
}
```

**Parameter Description**:
- **workspace_id**: The ID of the workspace to which the dataspace belongs, assigned by referencing the `id` of the workspace resource `huaweicloud_secmaster_workspace.test`
- **dataspace_name**: The dataspace name, assigned by referencing the input variable dataspace_name. The name can only contain English letters, digits and hyphens (-), and cannot start or end with a hyphen (-), nor can they appear consecutively. Valid length: 5-63
- **description**: The dataspace description, assigned by referencing the input variable dataspace_description

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
workspace_name = "secmaster-workspace-test"
dataspace_name = "secmaster-dataspace-test"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="workspace_name=secmaster-workspace-test"`
2. Environment variables: `export TF_VAR_workspace_name=secmaster-workspace-test`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the SecMaster dataspace
4. Run `terraform show` to view the created SecMaster dataspace

## Reference Information

- [Huawei Cloud SecMaster Product Documentation](https://support.huaweicloud.com/intl/en-us/secmaster/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For SecMaster Dataspace](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/dataspace)
