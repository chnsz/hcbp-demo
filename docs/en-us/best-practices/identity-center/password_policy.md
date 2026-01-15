# Deploy Password Policy

## Application Scenario

Identity Center is a unified identity management service provided by Huawei Cloud, supporting unified identity authentication and authorization management across clouds and applications. By configuring Identity Center password policies, you can set security policies such as maximum password validity period, minimum length, reuse prevention, character requirements, etc., improving account security and meeting enterprise-level security compliance requirements. This best practice will introduce how to use Terraform to automatically deploy Identity Center password policies, including querying or creating Identity Center instances, registering regions (optional), and configuring password policies.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Identity Center Instance Data Source (huaweicloud_identitycenter_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identitycenter_instance)

### Resources

- [Identity Center Registered Region Resource (huaweicloud_identitycenter_registered_region)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_registered_region)
- [Identity Center Instance Resource (huaweicloud_identitycenter_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_instance)
- [Identity Center Password Policy Resource (huaweicloud_identitycenter_password_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_password_policy)

### Resource/Data Source Dependencies

```text
huaweicloud_identitycenter_registered_region
    └── huaweicloud_identitycenter_instance
        └── huaweicloud_identitycenter_password_policy

data.huaweicloud_identitycenter_instance
    └── huaweicloud_identitycenter_password_policy
```

> Note: Identity Center password policy resource depends on Identity Center instance. If the instance does not exist, you need to create the instance first; if the instance already exists, you can query instance information through data source. When creating an instance, if you need to use it in a specific region, you may need to register the region first.

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

# Get Identity Center instance information in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), used for configuring password policy
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

### 5. Create Identity Center Password Policy Resource

Add the following script to the TF file (e.g., main.tf) to create Identity Center password policy:

```hcl
variable "policy_max_password_age" {
  description = "The max password age of the identity center password policy, unit in days"
  type        = number
  default     = 10

  validation {
    condition     = var.policy_max_password_age >= 1 && var.policy_max_password_age <= 1095
    error_message = "The valid value of the max password age is range from 1 to 1095"
  }
}

variable "policy_minimum_password_length" {
  description = "The minimum password length of the identity center password policy"
  type        = number
  default     = 10
}

variable "policy_password_reuse_prevention" {
  description = "The password reuse prevention feature of the identity center's password policy indicates whether to prohibit the use of the same password as the previous one"
  type        = bool
  default     = true
}

variable "policy_require_uppercase_characters" {
  description = "The require uppercase characters of the identity center password policy"
  type        = bool
  default     = true
}

variable "policy_require_lowercase_characters" {
  description = "The require lowercase characters of the identity center password policy"
  type        = bool
  default     = true
}

variable "policy_require_numbers" {
  description = "The require numbers of the identity center password policy"
  type        = bool
  default     = true
}

variable "policy_require_symbols" {
  description = "The require symbols of the identity center password policy"
  type        = bool
  default     = true
}

# Create Identity Center password policy resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identitycenter_password_policy" "test" {
  identity_store_id            = var.is_instance_create ? huaweicloud_identitycenter_instance.test[0].identity_store_id : data.huaweicloud_identitycenter_instance.test[0].identity_store_id
  max_password_age             = var.policy_max_password_age
  minimum_password_length      = var.policy_minimum_password_length
  password_reuse_prevention    = var.policy_password_reuse_prevention ? 1 : null
  require_uppercase_characters = var.policy_require_uppercase_characters
  require_lowercase_characters = var.policy_require_lowercase_characters
  require_numbers              = var.policy_require_numbers
  require_symbols              = var.policy_require_symbols
}
```

**Parameter Description**:
- **identity_store_id**: Identity store ID, assigned by referencing Identity Center instance resource or data source
- **max_password_age**: Maximum password validity period (days), assigned by referencing input variable policy_max_password_age, valid range is 1-1095, default value is 10
- **minimum_password_length**: Minimum password length, assigned by referencing input variable policy_minimum_password_length, default value is 10
- **password_reuse_prevention**: Password reuse prevention, assigned by referencing input variable policy_password_reuse_prevention, set to 1 to enable, set to null to disable, default value is 1 (enabled)
- **require_uppercase_characters**: Whether to require uppercase characters, assigned by referencing input variable policy_require_uppercase_characters, default value is true
- **require_lowercase_characters**: Whether to require lowercase characters, assigned by referencing input variable policy_require_lowercase_characters, default value is true
- **require_numbers**: Whether to require numbers, assigned by referencing input variable policy_require_numbers, default value is true
- **require_symbols**: Whether to require symbols, assigned by referencing input variable policy_require_symbols, default value is true

> Note: Identity Center password policy depends on Identity Center instance. If the instance does not exist, you need to create the instance first; if the instance already exists, you can query instance information through data source. Password policy parameters need to be configured reasonably according to actual security requirements. It is recommended to follow the principle of least privilege and set reasonable password complexity requirements to improve account security.

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Identity Center Password Policy Configuration
# The policy defines a minimum password length of 8 characters, allows uppercase and lowercase letters, numbers,
# and special characters, and a password validity period of 30 days.
policy_max_password_age        = 30
policy_minimum_password_length = 8
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially:
   - `is_instance_create` can be set to true to create instance, or set to false to use existing instance
   - `is_region_need_register` can be set to true to register region, or set to false to not register region
   - `instance_store_id_alias` can be set to instance alias, optional parameter
   - `policy_max_password_age` can be set to maximum password validity period (days), recommended to set to 30-90 days
   - `policy_minimum_password_length` can be set to minimum password length, recommended to set to 8 or longer
   - `policy_password_reuse_prevention` can be set to true to enable password reuse prevention, or set to false to disable
   - `policy_require_uppercase_characters` can be set to true to require uppercase characters, or set to false to not require
   - `policy_require_lowercase_characters` can be set to true to require lowercase characters, or set to false to not require
   - `policy_require_numbers` can be set to true to require numbers, or set to false to not require
   - `policy_require_symbols` can be set to true to require symbols, or set to false to not require
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="policy_minimum_password_length=12" -var="policy_max_password_age=90"`
2. Environment variables: `export TF_VAR_policy_minimum_password_length=12` and `export TF_VAR_policy_max_password_age=90`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values. Identity Center password policy depends on Identity Center instance. Please ensure the instance exists or is configured to be created. Password policy parameters need to be configured reasonably according to actual security requirements. It is recommended to follow the principle of least privilege.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create Identity Center password policy:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating Identity Center instance (if needed) and password policy
4. Run `terraform show` to view the details of the created Identity Center password policy

> Note: Before creating Identity Center password policy, ensure that Identity Center instance exists. If the instance does not exist, you need to create the instance first. When creating an instance, if you need to use it in a specific region, you may need to register the region first. It is recommended to confirm the status of Identity Center instance before creating password policy and configure password policy parameters reasonably according to actual security requirements.

## Reference Information

- [Huawei Cloud Identity Center Product Documentation](https://support.huaweicloud.com/identity-center/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Password Policy](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/identity-center/password-policy)
