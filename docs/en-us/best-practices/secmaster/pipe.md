# Deploy Data Pipe

## Application Scenario

SecMaster is a next-generation cloud native security operations center provided by Huawei Cloud, offering capabilities such as cloud asset management, security posture management, security information and event management, and security orchestration and automatic response. A data pipe is used to collect, partition, and index security data within a dataspace, serving as the foundation for security data analysis and retrieval.

This best practice will introduce how to use Terraform to automatically deploy a SecMaster data pipe, including workspace creation, dataspace creation, and the shard, storage period, and index mapping configuration of the data pipe.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [SecMaster Workspace (huaweicloud_secmaster_workspace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_workspace)
- [SecMaster Dataspace (huaweicloud_secmaster_dataspace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_dataspace)
- [SecMaster Data Pipe (huaweicloud_secmaster_pipe)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_pipe)

### Resource/Data Source Dependencies

```
huaweicloud_secmaster_workspace
    └── huaweicloud_secmaster_dataspace
            └── huaweicloud_secmaster_pipe
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a SecMaster Workspace

Add the following script in the TF file (such as main.tf):

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
- **project_name**: The project name of the workspace, assigned by referencing the input variable region_name
- **description**: The workspace description, assigned by referencing the input variable workspace_description

### 3. Create a SecMaster Dataspace

Add the following script in the TF file (such as main.tf):

```hcl
# Create a SecMaster dataspace resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dataspace_name" {
  description = "The name of the SecMaster dataspace"
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
- **workspace_id**: The ID of the workspace to which the dataspace belongs, assigned by referencing the ID of the workspace created in the previous step
- **dataspace_name**: The dataspace name, assigned by referencing the input variable dataspace_name
- **description**: The dataspace description, assigned by referencing the input variable dataspace_description

### 4. Create a SecMaster Data Pipe

Add the following script in the TF file (such as main.tf):

```hcl
# Create a SecMaster data pipe resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "pipe_name" {
  description = "The name of the data pipe"
  type        = string
}

variable "shards" {
  description = "The number of partitions for the data pipe. Range: 1-64"
  type        = number
  default     = 3
}

variable "storage_period" {
  description = "The data retention period in days. Range: 7-180"
  type        = number
  default     = 30
}

variable "pipe_description" {
  description = "The description of the data pipe"
  type        = string
  default     = "Created by Terraform"
}

variable "timestamp_field" {
  description = "The timestamp field for the data pipe"
  type        = string
  default     = "timestamp"
}

variable "pipe_status" {
  description = "The status of the pipe. Valid values: open, closed"
  type        = string
  default     = "open"
}

resource "huaweicloud_secmaster_pipe" "test" {
  workspace_id    = huaweicloud_secmaster_workspace.test.id
  dataspace_id    = huaweicloud_secmaster_dataspace.test.id
  pipe_name       = var.pipe_name
  shards          = var.shards
  storage_period  = var.storage_period
  description     = var.pipe_description
  timestamp_field = var.timestamp_field
  status          = var.pipe_status

  mapping = jsonencode({
    id   = {
      is_chinese_exist = true
      properties       = {}
      type             = "text"
    }
    name = {
      is_chinese_exist = false
      properties       = {}
      type             = "text"
    }
  })
}
```

**Parameter Description**:
- **workspace_id**: The ID of the workspace to which the data pipe belongs, assigned by referencing the ID of the workspace resource
- **dataspace_id**: The ID of the dataspace to which the data pipe belongs, assigned by referencing the ID of the dataspace resource
- **pipe_name**: The data pipe name, assigned by referencing the input variable pipe_name; it cannot be updated after creation
- **shards**: The number of partitions for the data pipe, assigned by referencing the input variable shards
- **storage_period**: The data retention period in days, assigned by referencing the input variable storage_period
- **description**: The data pipe description, assigned by referencing the input variable pipe_description
- **timestamp_field**: The timestamp field for the data pipe, assigned by referencing the input variable timestamp_field
- **status**: The data pipe status, assigned by referencing the input variable pipe_status
- **mapping**: The index mapping configuration, defined as a JSON string with field types and properties

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
workspace_name = "tf-workspace-test"
dataspace_name = "tf-dataspace-test"
pipe_name      = "tfpipe01"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="workspace_name=my-workspace"`
2. Environment variables: `export TF_VAR_workspace_name=my-workspace`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the data pipe
4. Run `terraform show` to view the created data pipe

## Reference Information

- [Huawei Cloud SecMaster Product Documentation](https://support.huaweicloud.com/intl/en-us/secmaster/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For SecMaster Data Pipe](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/pipe)
