# Deploy Fine-Grained Permission Management

## Application Scenario

Resource Access Manager (RAM) is a resource sharing service provided by Huawei Cloud, supporting cross-account resource sharing and management to help you achieve unified resource management and access control. Through fine-grained permission management, precise permission control can be configured for resource sharing, achieving more flexible and secure resource sharing. By querying available permissions and associating them with resource shares, the usage scope and operation permissions of shared resources can be precisely controlled. This best practice introduces how to use Terraform to automatically deploy fine-grained permission management, including querying available permissions and associating permissions with resource shares.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Resource Permissions Query Data Source (data.huaweicloud_ram_resource_permissions)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ram_resource_permissions)

### Resources

- [Resource Share Permission Resource (huaweicloud_ram_resource_share_permission)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ram_resource_share_permission)

### Resource/Data Source Dependencies

```text
data.huaweicloud_ram_resource_permissions
    └── huaweicloud_ram_resource_share_permission
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Query Available Resource Permissions

Add the following script to the TF file (such as main.tf) to query available resource permissions:

```hcl
variable "query_resource_type" {
  description = "The resource type for querying available permissions"
  type        = string
  default     = ""
}

variable "query_permission_type" {
  description = "The type of the permission to query"
  type        = string
  default     = "ALL"
}

variable "query_permission_name" {
  description = "The name of the permission to query"
  type        = string
  default     = ""
}

# Query available resource permissions
data "huaweicloud_ram_resource_permissions" "test" {
  resource_type   = var.query_resource_type != "" ? var.query_resource_type : null
  permission_type = var.query_permission_type
  name            = var.query_permission_name != "" ? var.query_permission_name : null
}
```

**Parameter Description**:
- **resource_type**: Resource type, assigned by referencing the input variable `query_resource_type`, used to specify the resource type for querying permissions, optional parameter
- **permission_type**: Permission type, assigned by referencing the input variable `query_permission_type`. Optional values include `ALL` (all), `SYSTEM` (system permissions), `CUSTOM` (custom permissions), default is `ALL`
- **name**: Permission name, assigned by referencing the input variable `query_permission_name`, used to filter permissions by name, optional parameter

> Note: By setting different query conditions, a list of permissions that meet the requirements can be filtered. If the resource type is not specified, permissions for all resource types will be queried.

### 3. Associate Permissions with Resource Share

Add the following script to the TF file (such as main.tf) to associate permissions with a resource share:

```hcl
variable "resource_share_id" {
  description = "The ID of the RAM resource share"
  type        = string
}

variable "permission_replace" {
  description = "Whether to replace existing permissions when associating a new permission"
  type        = bool
  default     = false
}

# Batch associate queried permissions with resource share
resource "huaweicloud_ram_resource_share_permission" "test" {
  count = length(data.huaweicloud_ram_resource_permissions.test.permissions)

  resource_share_id = var.resource_share_id
  permission_id     = data.huaweicloud_ram_resource_permissions.test.permissions[count.index].id
  replace           = var.permission_replace
}
```

**Parameter Description**:
- **count**: Resource count, dynamically creates resources based on the number of permissions queried
- **resource_share_id**: Resource share ID, assigned by referencing the input variable `resource_share_id`, used to specify the resource share to associate permissions with
- **permission_id**: Permission ID, assigned by referencing the permission ID in the data source query results
- **replace**: Whether to replace existing permissions, assigned by referencing the input variable `permission_replace`. When associating a new permission, whether to replace existing permissions, default is `false`

> Note: Through the `count` parameter, resources can be dynamically created based on the number of permissions queried, achieving batch permission association. If `replace` is `true`, existing permissions of the resource share will be replaced when associating a new permission; if `false`, new permissions will be appended.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Permission query configuration (Optional)
query_resource_type = "vpc:subnets"

# Resource share ID (Required)
resource_share_id = "your-resource-share-id"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="query_resource_type=vpc:subnets" -var="resource_share_id=your-resource-share-id"`
2. Environment variables: `export TF_VAR_resource_share_id=your-resource-share-id`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start associating permissions with the resource share
4. Run `terraform show` to view the associated permissions

## Reference Information

- [Huawei Cloud RAM Product Documentation](https://support.huaweicloud.com/ram/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Fine-Grained Permission Management](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ram/fine-grained-permission-management)
