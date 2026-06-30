# Deploy Data Recognition Rule Group

## Application Scenario

DataArts Studio provides a data security module that supports data classification and sensitivity level identification through data recognition rules. Data recognition rule groups are used to organize multiple data recognition rules together for unified management and application, forming an important part of the data security governance system.

This best practice is suitable for scenarios where you need to create data secrecy levels, data recognition rules, and data recognition rule groups in a DataArts Studio workspace, covering workspace query, data category query, data secrecy level creation, data recognition rule creation, rule group creation, and creation result verification. This best practice will introduce how to use Terraform to automatically deploy the above resources for Infrastructure as Code management of DataArts data security recognition rule groups.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [DataArts Studio Workspace List Query Data Source (data.huaweicloud_dataarts_studio_workspaces)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_studio_workspaces)
- [DataArts Data Category List Query Data Source (data.huaweicloud_dataarts_security_data_categories)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_security_data_categories)
- [DataArts Data Recognition Rule Group List Query Data Source (data.huaweicloud_dataarts_security_data_recognition_rule_groups)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_security_data_recognition_rule_groups)

### Resources

- [DataArts Data Secrecy Level Resource (huaweicloud_dataarts_security_data_secrecy_level)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_security_data_secrecy_level)
- [DataArts Data Recognition Rule Resource (huaweicloud_dataarts_security_data_recognition_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_security_data_recognition_rule)
- [DataArts Data Recognition Rule Group Resource (huaweicloud_dataarts_security_data_recognition_rule_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_security_data_recognition_rule_group)

### Resource/Data Source Dependencies

```
data.huaweicloud_dataarts_studio_workspaces
    ├── data.huaweicloud_dataarts_security_data_categories
    ├── huaweicloud_dataarts_security_data_secrecy_level
    ├── huaweicloud_dataarts_security_data_recognition_rule
    ├── huaweicloud_dataarts_security_data_recognition_rule_group
    └── data.huaweicloud_dataarts_security_data_recognition_rule_groups

data.huaweicloud_dataarts_security_data_categories
    └── huaweicloud_dataarts_security_data_recognition_rule

huaweicloud_dataarts_security_data_secrecy_level
    └── huaweicloud_dataarts_security_data_recognition_rule

huaweicloud_dataarts_security_data_recognition_rule
    └── huaweicloud_dataarts_security_data_recognition_rule_group

huaweicloud_dataarts_security_data_recognition_rule_group
    └── data.huaweicloud_dataarts_security_data_recognition_rule_groups
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Query DataArts Studio Workspaces

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query DataArts Studio workspace information and resolve the workspace ID. When workspace_id is specified, the workspace query data source can be skipped:

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

# Get DataArts Studio workspace information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create data recognition rule groups
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

locals {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
}
```

**Parameter Description**:
- **count**: Number of data sources to create, executes workspace query only when workspace_id is empty
- **instance_id**: DataArts Studio instance ID, assigned by referencing the input variable instance_id
- **name**: Workspace name, used to filter query results when workspace_name is not empty
- **workspace_id**: Workspace ID, used for exact query when workspace_id is not empty
- **lifecycle.precondition**: Precondition, instance_id must be provided when workspace_id is empty
- **local.workspace_id**: Local variable, uses workspace_id when it is not empty, otherwise references the return result of the workspace query data source

### 3. Query Data Categories

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query data category information in the workspace and resolve category IDs required for data recognition rules:

```hcl
variable "category_ids" {
  description = "The ID list of data categories used by data recognition rules"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "data_recognition_rule_count" {
  description = "The number of data recognition rules to create and include in the rule group"
  type        = number
  default     = 1
}

# Get data category information in the DataArts Studio workspace under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create data recognition rules
# Query data categories in the workspace to resolve category IDs for data recognition rules.
data "huaweicloud_dataarts_security_data_categories" "test" {
  workspace_id = local.workspace_id
}

locals {
  category_ids = length(var.category_ids) > 0 ? var.category_ids : slice([
    for category in data.huaweicloud_dataarts_security_data_categories.test.categories : category.category_id
  ], 0, var.data_recognition_rule_count)
}
```

**Parameter Description**:
- **workspace_id**: Workspace ID, referencing local variable local.workspace_id
- **category_ids**: Local variable, uses category_ids when it is not empty, otherwise slices the first data_recognition_rule_count category IDs from the data category query result
- **category_ids** (input variable): Data category ID list, assigned by referencing the input variable category_ids, default value is an empty list
- **data_recognition_rule_count**: Number of data recognition rules to create, assigned by referencing the input variable data_recognition_rule_count, default value is 1

### 4. Create Data Secrecy Levels

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create data secrecy level resources for data recognition rules:

```hcl
variable "rule_group_name" {
  description = "The name of the data recognition rule group"
  type        = string
  default     = ""
  nullable    = false
}

# Create DataArts data secrecy level resources under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) for data recognition rules
# Create data secrecy levels for data recognition rules.
resource "huaweicloud_dataarts_security_data_secrecy_level" "test" {
  count = var.data_recognition_rule_count

  workspace_id = local.workspace_id
  name         = format("%s_secrecy_level_%d", var.rule_group_name, count.index)
}
```

**Parameter Description**:
- **count**: Number of resources to create, equals data_recognition_rule_count
- **workspace_id**: Workspace ID, referencing local variable local.workspace_id
- **name**: Data secrecy level name, automatically generated based on rule_group_name and index

### 5. Create Data Recognition Rules

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create data recognition rule resources:

```hcl
# Create DataArts data recognition rule resources under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
# Create data recognition rules that will be grouped together.
resource "huaweicloud_dataarts_security_data_recognition_rule" "test" {
  count = var.data_recognition_rule_count

  workspace_id     = local.workspace_id
  rule_type        = "CUSTOM"
  name             = format("%s_rule_%d", var.rule_group_name, count.index)
  secrecy_level_id = huaweicloud_dataarts_security_data_secrecy_level.test[count.index].id
  category_id      = local.category_ids[count.index]
  method           = "NONE"

  lifecycle {
    precondition {
      condition     = length(local.category_ids) >= var.data_recognition_rule_count
      error_message = "The number of available category IDs must be greater than or equal to data_recognition_rule_count."
    }
  }
}
```

**Parameter Description**:
- **count**: Number of resources to create, equals data_recognition_rule_count
- **workspace_id**: Workspace ID, referencing local variable local.workspace_id
- **rule_type**: Rule type, set to "CUSTOM" for custom rules
- **name**: Rule name, automatically generated based on rule_group_name and index
- **secrecy_level_id**: Data secrecy level ID, referencing the data secrecy level resource ID at the corresponding index
- **category_id**: Data category ID, referencing the value at the corresponding index in local variable local.category_ids
- **method**: Recognition method, set to "NONE"
- **lifecycle.precondition**: Precondition, the number of available category IDs must be greater than or equal to data_recognition_rule_count

### 6. Create Data Recognition Rule Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a data recognition rule group resource:

```hcl
variable "rule_group_description" {
  description = "The description of the data recognition rule group"
  type        = string
  default     = ""
}

# Create a DataArts data recognition rule group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
# Create a data recognition rule group that contains the created rules.
resource "huaweicloud_dataarts_security_data_recognition_rule_group" "test" {
  depends_on = [
    huaweicloud_dataarts_security_data_recognition_rule.test,
  ]

  workspace_id = local.workspace_id
  name         = var.rule_group_name
  description  = var.rule_group_description
  rule_ids     = huaweicloud_dataarts_security_data_recognition_rule.test[*].id
}
```

**Parameter Description**:
- **depends_on**: Explicitly depends on data recognition rule resources to ensure rules are created before the rule group
- **workspace_id**: Workspace ID, referencing local variable local.workspace_id
- **name**: Rule group name, assigned by referencing the input variable rule_group_name
- **description**: Rule group description, assigned by referencing the input variable rule_group_description, default value is an empty string
- **rule_ids**: Rule ID list, referencing the IDs of all created data recognition rule resources

### 7. Query Data Recognition Rule Groups

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query the created data recognition rule group information to verify the rule group creation result:

```hcl
# Get DataArts data recognition rule group information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) to verify the rule group creation result
# Query the created rule group to verify the group metadata after creation.
data "huaweicloud_dataarts_security_data_recognition_rule_groups" "test" {
  depends_on = [
    huaweicloud_dataarts_security_data_recognition_rule_group.test,
  ]

  workspace_id = local.workspace_id
  name         = var.rule_group_name
}
```

**Parameter Description**:
- **depends_on**: Explicitly depends on the data recognition rule group resource to ensure the rule group is created before querying
- **workspace_id**: Workspace ID, referencing local variable local.workspace_id
- **name**: Rule group name, assigned by referencing the input variable rule_group_name, used to filter query results

### 8. Preset Input Parameters for Resource Deployment (Optional)

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

# Data recognition rule group
rule_group_name = "tf_test_rule_group"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the contents when executing terraform commands; other names require adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read variable values from this file

In addition to using a `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="rule_group_name=my-rule-group"`
2. Environment variables: `export TF_VAR_rule_group_name=my-rule-group`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform uses variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the data recognition rule group and related resources
4. Run `terraform show` to view details of the created data recognition rule group

## Reference Information

- [Huawei Cloud DataArts Studio Product Documentation](https://support.huaweicloud.com/intl/en-us/dataartsstudio/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DataArts Data Recognition Rule Group](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dataarts/security-data-recognition-rule-group)
