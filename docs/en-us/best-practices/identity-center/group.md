# Deploy Group

## Application Scenario

Identity Center is a unified identity management service provided by Huawei Cloud, supporting unified identity authentication and authorization management across clouds and applications. By creating groups, you can group multiple users for management, uniformly configure permission policies, perform access control, and conduct identity management, improving the efficiency and convenience of identity management. This best practice will introduce how to use Terraform to automatically deploy Identity Center groups, including querying or creating Identity Center instances, registering regions (optional), and creating groups.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Identity Center Instance Data Source (huaweicloud_identitycenter_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identitycenter_instance)

### Resources

- [Identity Center Registered Region Resource (huaweicloud_identitycenter_registered_region)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_registered_region)
- [Identity Center Instance Resource (huaweicloud_identitycenter_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_instance)
- [Identity Center Group Resource (huaweicloud_identitycenter_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_group)

### Resource/Data Source Dependencies

```text
huaweicloud_identitycenter_registered_region
    └── huaweicloud_identitycenter_instance
        └── huaweicloud_identitycenter_group

data.huaweicloud_identitycenter_instance
    └── huaweicloud_identitycenter_group
```

> Note: Identity Center group resource depends on Identity Center instance. If the instance does not exist, you need to create the instance first; if the instance already exists, you can query instance information through data source. When creating an instance, if you need to use it in a specific region, you may need to register the region first.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Identity Center Instance Data Source (Optional)

Add the following script to the TF file (e.g., main.tf) to query Identity Center instance information (when the instance already exists):

```hcl
variable "is_instance_create" {
  description = "Whether to create the identity center instance"
  type        = bool
  default     = true
}

# Get Identity Center instance information in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), used for creating groups
data "huaweicloud_identitycenter_instance" "test" {
  count = var.is_instance_create ? 0 : 1
}
```

**Parameter Description**:
- **count**: The number of data source creations, used to control whether to execute Identity Center instance query data source, only creates the data source (i.e., executes Identity Center instance query) when `var.is_instance_create` is false

### 3. Create Identity Center Registered Region Resource (Optional)

Add the following script to the TF file (e.g., main.tf) to register region (when instance needs to be created and region needs to be registered):

```hcl
variable "is_region_need_register" {
  description = "Whether to register the region"
  type        = bool
  default     = true
}

variable "region_name" {
  description = "The region where the LTS service is located"
  type        = string
}

# Create Identity Center registered region resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identitycenter_registered_region" "test" {
  count = var.is_instance_create ? var.is_region_need_register ? 1 : 0 : 0

  region_id = var.region_name
}
```

**Parameter Description**:
- **count**: Number of resource creations, used to control whether to create registered region resource, only creates when instance needs to be created and region needs to be registered
- **region_id**: Region ID, assigned by referencing input variable region_name

### 4. Create Identity Center Instance Resource (Optional)

Add the following script to the TF file (e.g., main.tf) to create Identity Center instance (when the instance does not exist):

```hcl
variable "instance_store_id_alias" {
  description = "The alias of the identity center instance"
  type        = string
  default     = ""
}

# Create Identity Center instance resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identitycenter_instance" "test" {
  count = var.is_instance_create ? 1 : 0

  depends_on = [huaweicloud_identitycenter_registered_region.test]

  alias = var.instance_store_id_alias != "" ? var.instance_store_id_alias : null
}
```

**Parameter Description**:
- **count**: Number of resource creations, used to control whether to create Identity Center instance resource, only creates when `var.is_instance_create` is true
- **depends_on**: Explicit dependency relationship, ensuring instance is created after registered region resource is created
- **alias**: Instance alias, assigned by referencing input variable instance_store_id_alias, optional parameter, default value is null

### 5. Create Identity Center Group Resource

Add the following script to the TF file (e.g., main.tf) to create Identity Center group:

```hcl
variable "group_name" {
  description = "The name of the identity center group"
  type        = string
}

variable "group_description" {
  description = "The description of the identity center group"
  type        = string
  default     = ""
}

# Create Identity Center group resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identitycenter_group" "test" {
  identity_store_id = var.is_instance_create ? huaweicloud_identitycenter_instance.test[0].identity_store_id : data.huaweicloud_identitycenter_instance.test[0].identity_store_id
  name              = var.group_name
  description       = var.group_description
}
```

**Parameter Description**:
- **identity_store_id**: Identity store ID, assigned by referencing Identity Center instance resource or data source
- **name**: Group name, assigned by referencing input variable group_name
- **description**: Group description, assigned by referencing input variable group_description, optional parameter, default value is empty string

> Note: Identity Center group depends on Identity Center instance. If the instance does not exist, you need to create the instance first; if the instance already exists, you can query instance information through data source. When creating an instance, if you need to use it in a specific region, you may need to register the region first.

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Identity Center Group Configuration
group_name = "tf_test_group"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially:
   - `is_instance_create` can be set to true to create instance, or set to false to use existing instance
   - `is_region_need_register` can be set to true to register region, or set to false to not register region
   - `instance_store_id_alias` can be set to instance alias, optional parameter
   - `group_name` needs to be set to group name
   - `group_description` can be set to group description information, optional parameter
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="group_name=my-group" -var="is_instance_create=true"`
2. Environment variables: `export TF_VAR_group_name=my-group` and `export TF_VAR_is_instance_create=true`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values. Identity Center group depends on Identity Center instance. Please ensure the instance exists or is configured to be created.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create Identity Center group:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating Identity Center instance (if needed) and group
4. Run `terraform show` to view the details of the created Identity Center group

> Note: Before creating Identity Center group, ensure that Identity Center instance exists. If the instance does not exist, you need to create the instance first. When creating an instance, if you need to use it in a specific region, you may need to register the region first. It is recommended to confirm the status of Identity Center instance before creating group.

## Reference Information

- [Huawei Cloud Identity Center Product Documentation](https://support.huaweicloud.com/identity-center/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Group](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/identity-center/group)
