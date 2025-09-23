# Deploy Cloud Application Server Group

## Application Scenario

Huawei Cloud Cloud Desktop (Workspace) is a cloud computing-based desktop virtualization service that provides enterprise users with secure and convenient cloud office solutions. Cloud application server groups are an important component of the Workspace service, used to host various applications and provide users with a unified application access experience.

Through cloud application server groups, enterprises can achieve centralized application deployment, unified management, and security control. Users can access cloud applications through various terminal devices without installing and maintaining software locally. This best practice will introduce how to use Terraform to automatically deploy Workspace cloud application server groups.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Workspace Service Query Data Source (data.huaweicloud_workspace_service)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_service)

### Resources

- [Workspace Application Server Group Resource (huaweicloud_workspace_app_server_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_app_server_group)

### Resource/Data Source Dependencies

```
data.huaweicloud_workspace_service.test
    └── huaweicloud_workspace_app_server_group.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Workspace Service Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the results of which are used to create cloud application server groups:

```hcl
# Get all Workspace service information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create cloud application server groups
data "huaweicloud_workspace_service" "test" {}
```

**Parameter Description**:
- This data source requires no additional parameters and automatically queries Workspace service information in the current region

### 3. Create Workspace Cloud Application Server Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a cloud application server group resource:

```hcl
# Create a Workspace cloud application server group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_workspace_app_server_group" "test" {
  name             = var.app_server_group_name
  app_type         = var.app_server_group_app_type
  os_type          = var.app_server_group_os_type
  flavor_id        = var.app_server_group_flavor_id
  image_type       = "gold"
  image_id         = var.app_server_group_image_id
  image_product_id = var.app_server_group_image_product_id
  vpc_id           = data.huaweicloud_workspace_service.test.vpc_id
  subnet_id        = try(data.huaweicloud_workspace_service.test.network_ids[0], null)
  system_disk_type = var.app_server_group_system_disk_type
  system_disk_size = var.app_server_group_system_disk_size
  is_vdi           = true
}
```

**Parameter Description**:
- **name**: Cloud application server group name, assigned by referencing the input variable app_server_group_name
- **app_type**: Application type, assigned by referencing the input variable app_server_group_app_type, default is "SESSION_DESKTOP_APP"
- **os_type**: Operating system type, assigned by referencing the input variable app_server_group_os_type, default is "Windows"
- **flavor_id**: Flavor ID, assigned by referencing the input variable app_server_group_flavor_id
- **image_type**: Image type, fixed as "gold" (golden image)
- **image_id**: Image ID, assigned by referencing the input variable app_server_group_image_id
- **image_product_id**: Image product ID, assigned by referencing the input variable app_server_group_image_product_id
- **vpc_id**: VPC ID, assigned based on the return results of the Workspace service query data source (data.huaweicloud_workspace_service)
- **subnet_id**: Subnet ID, assigned based on the return results of the Workspace service query data source (data.huaweicloud_workspace_service)
- **system_disk_type**: System disk type, assigned by referencing the input variable app_server_group_system_disk_type, default is "SAS"
- **system_disk_size**: System disk size, assigned by referencing the input variable app_server_group_system_disk_size, default is 80GB
- **is_vdi**: Whether it is VDI mode, fixed as true

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Cloud application server group basic information
app_server_group_name             = "tf_test_server_group"
app_server_group_app_type         = "SESSION_DESKTOP_APP"
app_server_group_os_type          = "Windows"
app_server_group_flavor_id        = "workspace.appstream.general.xlarge.4"
app_server_group_image_id         = "2ac7b1fb-b198-422b-a45f-61ea285cb6e7"
app_server_group_image_product_id = "OFFI886188719633408000"
app_server_group_system_disk_type = "SAS"
app_server_group_system_disk_size = 80
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="app_server_group_name=my-server-group" -var="app_server_group_flavor_id=workspace.appstream.general.xlarge.4"`
2. Environment variables: `export TF_VAR_app_server_group_name=my-server-group`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating cloud application server groups
4. Run `terraform show` to view the created cloud application server group

## Reference Information

- [Huawei Cloud Cloud Desktop Product Documentation](https://support.huaweicloud.com/workspace/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Workspace Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/app_server_group)
