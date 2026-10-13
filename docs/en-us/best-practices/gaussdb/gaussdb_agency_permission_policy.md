# Deploy Agency Permission Policy

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud. When using GaussDB, some features (such as disaster recovery and backup) rely on an agency to obtain permissions to access other cloud services, and the agency must have the corresponding permission policies bound to it in order to work properly.

This best practice will introduce how to use Terraform to automatically deploy a GaussDB agency permission policy, including agency name specification, permission policy binding, and permission policy unbinding.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [GaussDB Agency Permission Policy (huaweicloud_gaussdb_agency_permission_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_agency_permission_policy)

### Resource/Data Source Dependencies

```
huaweicloud_gaussdb_agency_permission_policy
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a GaussDB Agency Permission Policy

Add the following script in the TF file (such as main.tf):

```hcl
# Create a GaussDB agency permission policy resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "agency_name" {
  description = "The agency name. Currently, only RDSAccessProjectResource is supported"
  type        = string
  default     = "RDSAccessProjectResource"
}

variable "bind_role_names" {
  description = "The permission policies to be bound to the agency"
  type        = set(string)
}

variable "unbind_role_names" {
  description = "The permission policies to be unbound from the agency"
  type        = set(string)
}

resource "huaweicloud_gaussdb_agency_permission_policy" "test" {
  agency_name       = var.agency_name
  bind_role_names   = var.bind_role_names
  unbind_role_names = var.unbind_role_names
}
```

**Parameter Description**:

- **agency_name**: The agency name, assigned by referencing the input variable agency_name. Currently, only `RDSAccessProjectResource` is supported. This parameter is non-updatable and will trigger resource replacement if changed
- **bind_role_names**: The permission policies to be bound to the agency, assigned by referencing the input variable bind_role_names
- **unbind_role_names**: The permission policies to be unbound from the agency, assigned by referencing the input variable unbind_role_names

> Note: `bind_role_names` and `unbind_role_names` take effect in a single request. Deleting this resource only removes it from the Terraform state; the permission policy configuration on the agency remains unchanged.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Agency name (optional, defaults to RDSAccessProjectResource)
agency_name = "RDSAccessProjectResource"

# Permission policies to be bound to the agency
bind_role_names = ["DBS AgencyPolicy", "GaussDB FullAccess"]

# Permission policies to be unbound from the agency
unbind_role_names = ["GaussDB ReadOnlyAccess"]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="agency_name=RDSAccessProjectResource"`
2. Environment variables: `export TF_VAR_agency_name=RDSAccessProjectResource`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB agency permission policy
4. Run `terraform show` to view the created GaussDB agency permission policy

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Agency Permission Policy](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-agency-permission-policy)
