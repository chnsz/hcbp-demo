# Deploy Batch Associate Tags

## Application Scenario

Tag Management Service (TMS) is a tag management service provided by Huawei Cloud, supporting adding, modifying, and deleting tags for cloud resources, helping you achieve resource classification management and cost analysis. Through batch tag association, tags can be added to multiple cloud resources in batch, improving tag management efficiency. Batch tag association supports adding the same tags to different types of resources (such as ECS, VPC, RDS, etc.) in batch, achieving unified classification and identification of resources. This best practice introduces how to use Terraform to automatically deploy batch tag association, including querying project lists and batch associating tags.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Projects List Query Data Source (data.huaweicloud_identity_projects)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### Resources

- [Resource Tags Resource (huaweicloud_tms_resource_tags)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/tms_resource_tags)

### Resource/Data Source Dependencies

```text
data.huaweicloud_identity_projects
    └── huaweicloud_tms_resource_tags
```

> Note: Resource tags need to specify project ID to determine the scope of tags. Project information in the current region can be obtained by querying the projects list data source.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Query Projects List

Add the following script to the TF file (such as main.tf) to query the projects list:

```hcl
variable "region_name" {
  description = "The region where the LTS service is located"
  type        = string
}

# Query projects list data source
data "huaweicloud_identity_projects" "test" {
  name = var.region_name
}

# Get exact matching project ID through local values
locals {
  exact_project_id = try([for v in data.huaweicloud_identity_projects.test.projects : v.id if v.name == var.region_name][0], null)
}
```

**Parameter Description**:
- **name**: Project name, assigned by referencing the input variable `region_name`, used to filter projects
- **exact_project_id**: Exact matching project ID, filtered from the projects list through local values `locals` to get the project ID with matching name

> Note: The projects list is used to obtain project IDs. Project IDs need to be referenced when batch associating tags later to determine the scope of tags.

### 3. Batch Associate Tags

Add the following script to the TF file (such as main.tf) to batch associate tags:

```hcl
variable "associated_resources_configuration" {
  description = "The configuration of the associated resources"
  type        = list(object({
    type = string
    id   = string
  }))
}

variable "resource_tags" {
  description = "The tags of the resources"
  type        = map(string)
}

# Create resource tags resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_tms_resource_tags" "test" {
  project_id = local.exact_project_id

  dynamic "resources" {
    for_each = var.associated_resources_configuration

    content {
      resource_type = resources.value.type
      resource_id   = resources.value.id
    }
  }

  tags = var.resource_tags
}
```

**Parameter Description**:
- **project_id**: Project ID, assigned by referencing the local value `exact_project_id`, used to determine the scope of tags
- **resources**: Associated resource list, creates multiple resource configurations through dynamic block `dynamic "resources"` based on input variable `associated_resources_configuration`
  - **resource_type**: Resource type, assigned by referencing the `type` in the input variable, supports multiple resource types such as `dcs`, `ecs`, `vpc`
  - **resource_id**: Resource ID, assigned by referencing the `id` in the input variable
- **tags**: Tag key-value pairs, assigned by referencing the input variable `resource_tags`, format is `map(string)`

> Note: Batch tag association can add the same tags to multiple resources of different types in batch. Resource types and resource IDs need to match, ensuring resources exist. Tag key-value pairs can contain multiple tags for resource classification and identification.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Associated resources configuration (Required)
associated_resources_configuration = [
  {
    type = "dcs"
    id   = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  }
]

# Resource tags configuration (Required)
resource_tags = {
  foo   = "bar"
  owner = "terraform"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="associated_resources_configuration=[{type=\"dcs\",id=\"xxx\"}]" -var="resource_tags={foo=\"bar\"}"`
2. Environment variables: `export TF_VAR_resource_tags='{"foo":"bar","owner":"terraform"}'`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start batch associating tags
4. Run `terraform show` to view the associated tags

## Reference Information

- [Huawei Cloud TMS Product Documentation](https://support.huaweicloud.com/tms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Batch Associate Tags](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/tms/batch-associate-tags)
