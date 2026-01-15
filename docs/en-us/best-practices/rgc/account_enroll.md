# Deploy Account Enroll

## Application Scenario

Resource Governance Center (RGC) is a resource governance service provided by Huawei Cloud, supporting multi-account management, organizational unit management, blueprint configuration, and other functions to help you uniformly manage and govern cloud resources. Through the account enrollment function, accounts can be enrolled into organizational units and configured with blueprints to achieve automated resource deployment and management. This best practice introduces how to use Terraform to automatically deploy RGC account enrollment, including creating organizational units (optional) and account enrollment (with blueprint configuration).

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Organizational Unit Resource (huaweicloud_rgc_organizational_unit)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rgc_organizational_unit)
- [Account Enrollment Resource (huaweicloud_rgc_account_enroll)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rgc_account_enroll)

### Resource/Data Source Dependencies

```text
huaweicloud_rgc_organizational_unit
    └── huaweicloud_rgc_account_enroll
```

> Note: Creating an organizational unit is optional. If `create_organizational_unit` is `false`, the existing `parent_organizational_unit_id` will be used. Account enrollment requires specifying a parent organizational unit ID, which can reference a newly created organizational unit ID or use an existing organizational unit ID.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create Organizational Unit (Optional)

Add the following script to the TF file (such as main.tf) to create an organizational unit (optional):

```hcl
variable "organizational_unit_name" {
  description = "The name of the organizational unit to be created (required if create_organizational_unit is true)"
  type        = string
  default     = ""
}

variable "parent_organizational_unit_id" {
  description = "The ID of the parent organizational unit. Required for account enrollment and OU creation"
  type        = string
}

variable "create_organizational_unit" {
  description = "Whether to create a new organizational unit. If false, use existing parent_organizational_unit_id"
  type        = bool
  default     = true
}

# Create organizational unit resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_rgc_organizational_unit" "test" {
  count = var.create_organizational_unit ? 1 : 0

  organizational_unit_name      = var.organizational_unit_name
  parent_organizational_unit_id = var.parent_organizational_unit_id
}
```

**Parameter Description**:
- **count**: The number of resources to create, used to control whether to create an organizational unit. The organizational unit is only created when `var.create_organizational_unit` is `true`
- **organizational_unit_name**: The name of the organizational unit, assigned by referencing the input variable `organizational_unit_name`
- **parent_organizational_unit_id**: The ID of the parent organizational unit, assigned by referencing the input variable `parent_organizational_unit_id`

### 3. Create Account Enrollment

Add the following script to the TF file (such as main.tf) to create account enrollment (with blueprint configuration):

```hcl
variable "blueprint_managed_account_id" {
  description = "The ID of the account to be enrolled with blueprint configuration"
  type        = string
}

variable "blueprint_product_id" {
  description = "The ID of the blueprint product"
  type        = string
}

variable "blueprint_product_version" {
  description = "The version of the blueprint product"
  type        = string
}

variable "blueprint_variables" {
  description = "The variables for the blueprint configuration (JSON string format)"
  type        = string
}

variable "is_blueprint_has_multi_account_resource" {
  description = "Whether the blueprint has multi-account resources"
  type        = bool
}

# Create account enrollment resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_rgc_account_enroll" "test" {
  managed_account_id            = var.blueprint_managed_account_id
  parent_organizational_unit_id = var.create_organizational_unit ? huaweicloud_rgc_organizational_unit.test[0].organizational_unit_id : var.parent_organizational_unit_id

  blueprint {
    blueprint_product_id                    = var.blueprint_product_id
    blueprint_product_version               = var.blueprint_product_version
    variables                               = var.blueprint_variables
    is_blueprint_has_multi_account_resource = var.is_blueprint_has_multi_account_resource
  }
}
```

**Parameter Description**:
- **managed_account_id**: The ID of the account to be enrolled, assigned by referencing the input variable `blueprint_managed_account_id`
- **parent_organizational_unit_id**: The ID of the parent organizational unit. If an organizational unit is created, it references the newly created organizational unit ID; otherwise, it uses the existing organizational unit ID
- **blueprint**: Blueprint configuration block, containing the following parameters:
  - **blueprint_product_id**: Blueprint product ID, assigned by referencing the input variable `blueprint_product_id`
  - **blueprint_product_version**: Blueprint product version, assigned by referencing the input variable `blueprint_product_version`
  - **variables**: Blueprint variables, assigned by referencing the input variable `blueprint_variables`, must be in JSON string format
  - **is_blueprint_has_multi_account_resource**: Whether the blueprint contains multi-account resources, assigned by referencing the input variable `is_blueprint_has_multi_account_resource`

> Note: Blueprint variables must be in JSON string format, for example: `"{\"environment\":\"production\",\"region\":\"cn-north-4\"}"`. If the blueprint contains multi-account resources, set `is_blueprint_has_multi_account_resource` to `true`.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Parent organizational unit ID (Required)
# Required for both account enrollment and OU creation
# Replace with your actual parent organizational unit ID
parent_organizational_unit_id = "ou-xxxxxxxxxxxxx"

# Account ID to be enrolled (Required)
# Replace with your actual account ID
blueprint_managed_account_id = "account-xxxxxxxxxxxxx"

# Blueprint product configuration (Required)
# Replace with your actual blueprint product ID and version
blueprint_product_id      = "blueprint-xxxxxxxxxxxxx"
blueprint_product_version = "1.0.0"

# Blueprint variables (Required)
# Customize these variables according to your blueprint requirements
blueprint_variables = "{\"environment\":\"production\",\"region\":\"cn-north-4\"}"

# Whether the blueprint has multi-account resources (Required)
is_blueprint_has_multi_account_resource = false
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="parent_organizational_unit_id=ou-xxxxxxxxxxxxx" -var="blueprint_managed_account_id=account-xxxxxxxxxxxxx"`
2. Environment variables: `export TF_VAR_parent_organizational_unit_id=ou-xxxxxxxxxxxxx`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating account enrollment
4. Run `terraform show` to view the created account enrollment

## Reference Information

- [Huawei Cloud RGC Product Documentation](https://support.huaweicloud.com/rgc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Account Enrollment](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rgc/account-enroll)
