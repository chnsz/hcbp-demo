# Deploy DataArts Studio Workspace User

## Application Scenario

DataArts Studio is a one-stop data operations and governance platform provided by Huawei Cloud. A workspace is the basic unit for data development and management in DataArts Studio. By adding users to a workspace and assigning roles, you can enable multi-user collaborative development and implement fine-grained permission control for different users.

This best practice is suitable for scenarios where you need to add IAM users to a DataArts Studio workspace and assign roles, covering workspace query, workspace user role query, IAM user query, and workspace user creation. This best practice will introduce how to use Terraform to automatically deploy the above resources for Infrastructure as Code management of DataArts Studio workspace users.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [DataArts Studio Workspace List Query Data Source (data.huaweicloud_dataarts_studio_workspaces)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_studio_workspaces)
- [DataArts Studio Workspace User Role List Query Data Source (data.huaweicloud_dataarts_studio_workspace_user_roles)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_studio_workspace_user_roles)
- [IAM User List Query Data Source (data.huaweicloud_identity_users)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_users)

### Resources

- [DataArts Studio Workspace User Resource (huaweicloud_dataarts_studio_workspace_user)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_studio_workspace_user)

### Resource/Data Source Dependencies

```
data.huaweicloud_dataarts_studio_workspaces
    ├── data.huaweicloud_dataarts_studio_workspace_user_roles
    └── huaweicloud_dataarts_studio_workspace_user

data.huaweicloud_dataarts_studio_workspace_user_roles
    └── huaweicloud_dataarts_studio_workspace_user

data.huaweicloud_identity_users
    └── huaweicloud_dataarts_studio_workspace_user
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Query DataArts Studio Workspaces

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query DataArts Studio workspace information. Skip this step when workspace_id is specified:

```hcl
variable "workspace_id" {
  description = "The ID of the workspace"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_id" {
  description = "The ID of the DataArts Studio instance to which the workspace belongs"
  type        = string
  default     = ""
  nullable    = false
}

variable "workspace_name" {
  description = "The name of the workspace used to filter results"
  type        = string
  default     = ""
  nullable    = false
}

# Get DataArts Studio workspace information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create workspace users
# Query workspaces under the specified DataArts Studio instance.
data "huaweicloud_dataarts_studio_workspaces" "test" {
  count = var.workspace_id == "" ? 1 : 0

  instance_id  = var.instance_id
  name         = var.workspace_name != "" ? var.workspace_name : null
  workspace_id = var.workspace_id != "" ? var.workspace_id : null

  lifecycle {
    precondition {
      condition     = var.instance_id != ""
      error_message = "instance_id must be provided if workspace_id is omitted."
    }
  }
}
```

**Parameter Description**:
- **count**: Number of data sources to create, executes workspace query only when workspace_id is empty
- **instance_id**: DataArts Studio instance ID, assigned by referencing the input variable instance_id
- **name**: Workspace name, used to filter query results when workspace_name is not empty
- **workspace_id**: Workspace ID, used for exact query when workspace_id is not empty
- **lifecycle.precondition**: Precondition, instance_id must be provided when workspace_id is empty

### 3. Query Workspace User Roles

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query available user roles under the target workspace:

```hcl
# Get DataArts Studio workspace user role information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create workspace users
# Query available workspace user roles under the target workspace.
data "huaweicloud_dataarts_studio_workspace_user_roles" "test" {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
}
```

**Parameter Description**:
- **workspace_id**: Workspace ID, uses workspace_id when it is not empty, otherwise references the return result of the workspace query data source

### 4. Query IAM Users

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query IAM user information by name. Skip this step when user_id is specified:

```hcl
variable "user_id" {
  description = "The ID of the IAM user to be added to the workspace"
  type        = string
  default     = ""
  nullable    = false
}

variable "user_name" {
  description = "The name of the IAM user to be added to the workspace"
  type        = string
  default     = ""
  nullable    = false
}

# Get IAM user information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create workspace users
# Query the IAM user by name when user_id is omitted.
data "huaweicloud_identity_users" "test" {
  count = var.user_id == "" ? 1 : 0

  name = var.user_name

  lifecycle {
    precondition {
      condition     = var.user_name != ""
      error_message = "user_name must be provided if user_id is omitted."
    }
  }
}
```

**Parameter Description**:
- **count**: Number of data sources to create, executes IAM user query only when user_id is empty
- **name**: IAM user name, assigned by referencing the input variable user_name
- **lifecycle.precondition**: Precondition, user_name must be provided when user_id is empty

### 5. Create DataArts Studio Workspace User

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a workspace user and assign roles:

```hcl
variable "role_ids" {
  description = "The role ID list of the workspace user"
  type        = list(string)
  default     = []
  nullable    = false
}

# Create a DataArts Studio workspace user resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
# Create a workspace user and assign roles.
resource "huaweicloud_dataarts_studio_workspace_user" "test" {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
  user_id      = var.user_id != "" ? var.user_id : try(data.huaweicloud_identity_users.test[0].users[0].id, "")

  dynamic "roles" {
    for_each = var.role_ids

    content {
      id = roles.value
    }
  }

  lifecycle {
    precondition {
      condition = length([
        for role_id in var.role_ids : role_id
        if contains([for role in data.huaweicloud_dataarts_studio_workspace_user_roles.test.roles : role.id], role_id)
      ]) == length(var.role_ids)
      error_message = "All role_ids must exist in the workspace user roles returned by the data source."
    }
  }
}
```

**Parameter Description**:
- **workspace_id**: Workspace ID, uses workspace_id when it is not empty, otherwise references the return result of the workspace query data source
- **user_id**: IAM user ID, uses user_id when it is not empty, otherwise references the return result of the IAM user query data source
- **roles.id**: Workspace user role ID, assigned by referencing the input variable role_ids
- **lifecycle.precondition**: Precondition, all role IDs in role_ids must exist in the return result of the workspace user role query data source

### 6. Preset Input Parameters for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration values. These input parameters need to be entered manually during subsequent deployment.
Terraform also provides a method to preset these configurations through a `terraform.tfvars` file to avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# DataArts Studio workspace
workspace_id = "your-dataarts-studio-workspace-id"

# Workspace user
user_name = "your-iam-user-name"
role_ids  = ["r00001"]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the contents when executing terraform commands; other names require adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read variable values from this file

In addition to using a `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="user_name=my-user" -var='role_ids=["r00001"]'`
2. Environment variables: `export TF_VAR_user_name=my-user`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform uses variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the workspace user
4. Run `terraform show` to view details of the created workspace user

## Reference Information

- [Huawei Cloud DataArts Studio Product Documentation](https://support.huaweicloud.com/intl/en-us/dataartsstudio/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DataArts Studio Workspace User](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dataarts/studio-workspace-user)
