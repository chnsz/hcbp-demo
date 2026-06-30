# Deploy DataArts Factory Script Execute

## Application Scenario

DataArts Studio is a one-stop data operations and governance platform provided by Huawei Cloud. The DataArts Factory (data development) module provides a fully managed big data scheduling and development environment, supporting the creation and execution of script types such as DLI SQL. Through Factory scripts, users can write and manage data development scripts in workspaces and associate DLI queues, databases, and other resources to complete data processing tasks.

This best practice is suitable for scenarios where you need to create a DLI SQL script in a DataArts Studio workspace and execute it once, covering workspace query, DLI data connection query, DLI database and table creation, Factory script creation, and script execution. This best practice will introduce how to use Terraform to automatically deploy the above resources for Infrastructure as Code management of DataArts Factory script development and execution.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [DataArts Studio Workspace List Query Data Source (data.huaweicloud_dataarts_studio_workspaces)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_studio_workspaces)
- [DataArts Studio Data Connection List Query Data Source (data.huaweicloud_dataarts_studio_data_connections)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_studio_data_connections)

### Resources

- [DLI Database Resource (huaweicloud_dli_database)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_database)
- [DLI Table Resource (huaweicloud_dli_table)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_table)
- [DataArts Factory Script Resource (huaweicloud_dataarts_factory_script)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_factory_script)
- [DataArts Factory Script Execute Resource (huaweicloud_dataarts_factory_script_execute)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_factory_script_execute)

### Resource/Data Source Dependencies

```
data.huaweicloud_dataarts_studio_workspaces
    ├── data.huaweicloud_dataarts_studio_data_connections
    ├── huaweicloud_dataarts_factory_script
    └── huaweicloud_dataarts_factory_script_execute

data.huaweicloud_dataarts_studio_data_connections
    └── huaweicloud_dataarts_factory_script

huaweicloud_dli_database
    ├── huaweicloud_dli_table
    └── huaweicloud_dataarts_factory_script

huaweicloud_dli_table
    └── huaweicloud_dataarts_factory_script

huaweicloud_dataarts_factory_script
    └── huaweicloud_dataarts_factory_script_execute
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Query DataArts Studio Workspaces

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to Query DataArts Studio workspace information. Skip this step when workspace_id is specified:

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

# Get DataArts Studio workspace information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create Factory scripts and execute scripts
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

### 3. Query DLI Data Connections

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to Query DLI data connection information in the DataArts Studio workspace. Skip this step when connection_name is specified:

```hcl
variable "connection_name" {
  description = "The name of the DLI data connection"
  type        = string
  default     = ""
  nullable    = false
}

# Get DLI data connection information in the DataArts Studio workspace under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create Factory scripts
# Query data connections in the workspace to obtain the DLI connection name.
data "huaweicloud_dataarts_studio_data_connections" "test" {
  count = var.connection_name == "" ? 1 : 0

  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
  type         = "DLI"
}
```

**Parameter Description**:
- **count**: Number of data sources to create, executes data connection query only when connection_name is empty
- **workspace_id**: Workspace ID, uses workspace_id when it is not empty, otherwise references the return result of the workspace query data source
- **type**: Data connection type, set to "DLI" for DLI data connections

### 4. Create DLI Database

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to Create a DLI database resource as the data source for the Factory script:

```hcl
variable "dli_database_name" {
  description = "The name of the DLI database created for the script"
  type        = string
}

variable "dli_database_description" {
  description = "The description of the DLI database"
  type        = string
  default     = ""
}

# Create a DLI database resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) as the data source for the Factory script
# Create a DLI database as the data source for the script.
resource "huaweicloud_dli_database" "test" {
  name        = var.dli_database_name
  description = var.dli_database_description
}
```

**Parameter Description**:
- **name**: DLI database name, assigned by referencing the input variable dli_database_name
- **description**: DLI database description, assigned by referencing the input variable dli_database_description, default value is an empty string

### 5. Create DLI Table

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to Create a DLI table resource as the data source for the Factory script:

```hcl
variable "dli_table_name" {
  description = "The name of the DLI table created for the script"
  type        = string
}

variable "dli_table_description" {
  description = "The description of the DLI table"
  type        = string
  default     = ""
}

variable "dli_table_columns" {
  description = "The column definitions of the DLI table"
  type = list(object({
    name = string
    type = string
  }))

  default = [
    {
      name = "name"
      type = "string"
    },
    {
      name = "age"
      type = "int"
    },
  ]
}

# Create a DLI table resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) as the data source for the Factory script
# Create a DLI table as the data source for the script.
resource "huaweicloud_dli_table" "test" {
  database_name = huaweicloud_dli_database.test.name
  name          = var.dli_table_name
  data_location = "DLI"
  description   = var.dli_table_description

  dynamic "columns" {
    for_each = var.dli_table_columns

    content {
      name = columns.value.name
      type = columns.value.type
    }
  }
}
```

**Parameter Description**:
- **database_name**: DLI database name, referencing the name of the previously created DLI database resource
- **name**: DLI table name, assigned by referencing the input variable dli_table_name
- **data_location**: Data storage location, set to "DLI"
- **description**: DLI table description, assigned by referencing the input variable dli_table_description, default value is an empty string
- **columns**: Table column definitions, assigned by referencing the input variable dli_table_columns, default includes name (string) and age (int) columns

### 6. Create DataArts Factory Script

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to Create a DataArts Factory script resource:

```hcl
variable "script_name" {
  description = "The name of the DataArts Factory script"
  type        = string
}

variable "script_type" {
  description = "The type of the DataArts Factory script"
  type        = string
  default     = "DLISQL"
}

variable "script_directory" {
  description = "The directory path where the script is stored in DataArts Factory"
  type        = string
  default     = "/terraform"
}

variable "queue_name" {
  description = "The DLI queue name associated with the script"
  type        = string
  default     = "default"
}

variable "script_description" {
  description = "The description of the DataArts Factory script"
  type        = string
  default     = ""
}

variable "script_configuration" {
  description = "The user-defined configuration parameters of the DataArts Factory script"
  type        = map(string)
  default     = {}
}

variable "script_content" {
  description = "The SQL content of the DataArts Factory script"
  type        = string
  default     = ""
  nullable    = false
}

# Create a DataArts Factory script resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
# Create a DataArts Factory script that queries the DLI table.
resource "huaweicloud_dataarts_factory_script" "test" {
  depends_on = [
    huaweicloud_dli_database.test,
    huaweicloud_dli_table.test,
  ]

  workspace_id    = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
  name            = var.script_name
  type            = var.script_type
  connection_name = var.connection_name != "" ? var.connection_name : try(data.huaweicloud_dataarts_studio_data_connections.test[0].connections[0].name, "")
  directory       = var.script_directory
  database        = huaweicloud_dli_database.test.name
  queue_name      = var.queue_name
  description     = var.script_description
  configuration   = var.script_configuration
  content         = var.script_content != "" ? var.script_content : format("SELECT * FROM %s.%s", var.dli_database_name, var.dli_table_name)
}
```

**Parameter Description**:
- **depends_on**: Explicitly depends on DLI database and DLI table resources to ensure data sources are created before the script
- **workspace_id**: Workspace ID, uses workspace_id when it is not empty, otherwise references the return result of the workspace query data source
- **name**: Script name, assigned by referencing the input variable script_name
- **type**: Script type, assigned by referencing the input variable script_type, default value is "DLISQL"
- **connection_name**: DLI data connection name, uses connection_name when it is not empty, otherwise references the return result of the data connection query data source
- **directory**: Script storage directory, assigned by referencing the input variable script_directory, default value is "/terraform"
- **database**: Associated DLI database name, referencing the name of the previously created DLI database resource
- **queue_name**: Associated DLI queue name, assigned by referencing the input variable queue_name, default value is "default"
- **description**: Script description, assigned by referencing the input variable script_description
- **configuration**: User-defined configuration parameters, assigned by referencing the input variable script_configuration, default value is an empty map
- **content**: Script SQL content, uses script_content when it is not empty, otherwise automatically generates a SELECT statement querying the DLI table

### 7. Execute DataArts Factory Script

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to Execute the DataArts Factory script:

```hcl
variable "script_execute_params" {
  description = "The execution parameters passed to the script content, in key-value format"
  type        = map(string)
  default = {
    "spark.sql.adaptive.enabled"                                   = "true"
    "spark.sql.adaptive.join.enabled"                              = "true"
    "spark.sql.adaptive.join.skewedJoin.enabled"                   = "true"
    "spark.sql.forcePartitionPredicatesOnPartitionedTable.enabled" = "true"
    "spark.sql.mergeSmallFiles.enabled"                            = "true"
  }
}

# Execute a DataArts Factory script under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
# Execute the DataArts Factory script once.
resource "huaweicloud_dataarts_factory_script_execute" "test" {
  depends_on = [
    huaweicloud_dataarts_factory_script.test,
  ]

  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
  script_name  = huaweicloud_dataarts_factory_script.test.name
  params       = length(var.script_execute_params) > 0 ? jsonencode(var.script_execute_params) : null
}
```

**Parameter Description**:
- **depends_on**: Explicitly depends on the Factory script resource to ensure the script is created before execution
- **workspace_id**: Workspace ID, uses workspace_id when it is not empty, otherwise references the return result of the workspace query data source
- **script_name**: Script name, referencing the name of the previously created Factory script resource
- **params**: Script execution parameters, encoded as a JSON string via jsonencode when script_execute_params is not empty, otherwise null

### 8. Preset Input Parameters for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration values. These input parameters need to be entered manually during subsequent deployment.
Terraform also provides a method to preset these configurations through a `terraform.tfvars` file to avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# DataArts Studio instance and workspace
workspace_id = "your-dataarts-studio-workspace-id"

# DLI database and table
dli_database_name = "tf_test_database"
dli_table_name    = "tf_test_table"

# DataArts Factory script
script_name = "tf_test_factory_script"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the contents when executing terraform commands; other names require adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read variable values from this file

In addition to using a `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="script_name=my-script" -var="dli_database_name=my-db"`
2. Environment variables: `export TF_VAR_script_name=my-script`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform uses variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating and executing the DataArts Factory script
4. Run `terraform show` to view details of the created DataArts Factory script and execution

## Reference Information

- [Huawei Cloud DataArts Studio Product Documentation](https://support.huaweicloud.com/intl/en-us/dataartsstudio/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DataArts Factory Script Execute](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dataarts/factory-script-execute)
