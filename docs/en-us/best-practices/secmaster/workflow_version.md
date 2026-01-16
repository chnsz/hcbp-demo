# Deploy Workflow Version

## Application Scenario

Security Master (SecMaster) is a security situation awareness and security operations platform provided by Huawei Cloud, supporting unified management, analysis, and response of security events, helping you achieve automation and intelligence in security operations. Through workflow version functionality, different versions can be created for workflows, achieving workflow version management and iterative updates. Workflow versions include Base64-encoded workflow topology diagrams and parameter configurations, supporting JSON-format task flow definitions. This best practice introduces how to use Terraform to automatically deploy workflow versions, including workspace query, workflow query, and workflow version creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Workspace Query Data Source (data.huaweicloud_secmaster_workspaces)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/secmaster_workspaces)
- [Workflow Query Data Source (data.huaweicloud_secmaster_workflows)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/secmaster_workflows)

### Resources

- [Workflow Version Resource (huaweicloud_secmaster_workflow_version)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_workflow_version)

### Resource/Data Source Dependencies

```text
data.huaweicloud_secmaster_workspaces
    └── data.huaweicloud_secmaster_workflows
        └── huaweicloud_secmaster_workflow_version
```

> Note: Workspace and workflow queries are optional. If `workspace_id` and `workflow_id` are provided, these IDs are used directly; otherwise, the corresponding IDs are queried by name.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Query Workspace (Optional)

Add the following script to the TF file (such as main.tf) to query workspace (optional):

```hcl
variable "workspace_id" {
  description = "The ID of the workspace"
  type        = string
  default     = ""
  nullable    = false
}

variable "workspace_name" {
  description = "The name of the workspace"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.workspace_id != "" || var.workspace_name != ""
    error_message = "At least one of workspace_id and workspace_name must be provided."
  }
}

# Query workspace (query by name when workspace_id is empty)
data "huaweicloud_secmaster_workspaces" "test" {
  count = var.workspace_id == "" ? 1 : 0

  name = var.workspace_name
}
```

**Parameter Description**:
- **count**: Data source count, creates data source for query when `workspace_id` is empty
- **name**: Workspace name, assigned by referencing the input variable `workspace_name`

> Note: If `workspace_id` is provided, workspace query is not needed; if only `workspace_name` is provided, workspace ID needs to be queried through data source.

### 3. Query Workflow (Optional)

Add the following script to the TF file (such as main.tf) to query workflow (optional):

```hcl
variable "workflow_id" {
  description = "The ID of the workflow"
  type        = string
  default     = ""
  nullable    = false
}

variable "workflow_name" {
  description = "The name of the workflow"
  type        = string
}

# Query workflow (query by name when workflow_id is empty)
data "huaweicloud_secmaster_workflows" "test" {
  count = var.workflow_id == "" ? 1 : 0

  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  name         = var.workflow_name
}
```

**Parameter Description**:
- **count**: Data source count, creates data source for query when `workflow_id` is empty
- **workspace_id**: Workspace ID, prioritizes the input `workspace_id`, uses queried workspace ID if empty
- **name**: Workflow name, assigned by referencing the input variable `workflow_name`

> Note: If `workflow_id` is provided, workflow query is not needed; if only `workflow_name` is provided, workflow ID needs to be queried through data source.

### 4. Create Workflow Version

Add the following script to the TF file (such as main.tf) to create workflow version:

```hcl
variable "workflow_version_taskflow" {
  description = "The Base64 encoded of the workflow topology diagram"
  type        = string
}

variable "workflow_version_taskconfig" {
  description = "The parameters configuration of the workflow topology diagram"
  type        = string
}

variable "workflow_version_taskflow_type" {
  description = "The taskflow type of the workflow"
  type        = string
  default     = "JSON"
}

variable "workflow_version_aop_type" {
  description = "The aop type of the workflow"
  type        = string
  default     = "NORMAL"
}

variable "workflow_version_description" {
  description = "The description of the workflow version"
  type        = string
  default     = ""
}

# Create workflow version resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_secmaster_workflow_version" "test" {
  workspace_id  = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  workflow_id    = var.workflow_id != "" ? var.workflow_id : try(data.huaweicloud_secmaster_workflows.test[0].workflows[0].id, null)
  name           = var.workflow_name
  taskflow       = var.workflow_version_taskflow
  taskconfig     = var.workflow_version_taskconfig
  taskflow_type  = var.workflow_version_taskflow_type
  aop_type       = var.workflow_version_aop_type
  description    = var.workflow_version_description
}
```

**Parameter Description**:
- **workspace_id**: Workspace ID, prioritizes the input `workspace_id`, uses queried workspace ID if empty
- **workflow_id**: Workflow ID, prioritizes the input `workflow_id`, uses queried workflow ID if empty
- **name**: Workflow name, assigned by referencing the input variable `workflow_name`
- **taskflow**: Base64-encoded workflow topology diagram, assigned by referencing the input variable `workflow_version_taskflow`
- **taskconfig**: Parameter configuration of the workflow topology diagram, assigned by referencing the input variable `workflow_version_taskconfig`
- **taskflow_type**: Task flow type, assigned by referencing the input variable `workflow_version_taskflow_type`, default is `JSON`
- **aop_type**: AOP type, assigned by referencing the input variable `workflow_version_aop_type`, default is `NORMAL`
- **description**: Workflow version description, assigned by referencing the input variable `workflow_version_description`, optional parameter

> Note: Workflow topology diagrams need to be provided in Base64-encoded format. Task flow types are usually in JSON format, and AOP types default to NORMAL.

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Workspace and workflow information (Required)
workspace_name = "tf_test_workspace"
workflow_name  = "tf_test_workflow"

# Workflow version configuration (Required)
workflow_version_taskflow    = "your_workflow_taskflow"
workflow_version_taskconfig  = "your_workflow_taskconfig"
workflow_version_description = "This is a workflow version created by Terraform"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="workspace_name=tf_test_workspace" -var="workflow_name=tf_test_workflow"`
2. Environment variables: `export TF_VAR_workspace_name=tf_test_workspace`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the workflow version
4. Run `terraform show` to view the created workflow version

## Reference Information

- [Huawei Cloud SecMaster Product Documentation](https://support.huaweicloud.com/secmaster/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Workflow Version](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/workflow-version)
