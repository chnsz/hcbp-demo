# Deploy Password Policy

## Application Scenario

Identity and Access Management (IAM) is a basic identity authentication and access management service provided by Huawei Cloud, providing core functions such as identity management, permission management, and access control for Huawei Cloud users. By configuring password policies, you can set security policies such as password complexity requirements, validity periods, reuse rules, etc., improving account security and meeting enterprise-level security compliance requirements. This best practice will introduce how to use Terraform to automatically deploy IAM password policies, including configuration of security policies such as password length, character combination, validity period, reuse rules, etc.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [IAM Password Policy Resource (huaweicloud_identityv5_password_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identityv5_password_policy)

### Resource/Data Source Dependencies

```text
huaweicloud_identityv5_password_policy
```

> Note: IAM password policy resource is a global resource used to configure password security policies for IAM accounts. After password policy is configured, it will apply to all IAM users.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create IAM Password Policy Resource

Add the following script to the TF file (e.g., main.tf) to create IAM password policy:

```hcl
variable "policy_max_consecutive_identical_chars" {
  description = "The maximum number of times that a character is allowed to consecutively present in a password"
  type        = number

  validation {
    condition     = var.policy_max_consecutive_identical_chars >= 0 && var.policy_max_consecutive_identical_chars <= 32
    error_message = "The valid value of the maximum number of times that a character is allowed to consecutively present in a password is range from 0 to 32"
  }
}

variable "policy_min_password_age" {
  description = "The minimum period (minutes) after which users are allowed to make a password change"
  type        = number

  validation {
    condition     = var.policy_min_password_age >= 0 && var.policy_min_password_age <= 1440
    error_message = "The valid value of the minimum period (minutes) after which users are allowed to make a password change is range from 0 to 1440"
  }
}

variable "policy_min_password_length" {
  description = "The minimum number of characters that a password must contain"
  type        = number

  validation {
    condition     = var.policy_min_password_length >= 8 && var.policy_min_password_length <= 32
    error_message = "The valid value of the minimum number of characters that a password must contain is range from 8 to 32"
  }
}

variable "policy_password_reuse_prevention" {
  description = "The password reuse prevention feature of the identity center's password policy indicates whether to prohibit the use of the same password as the previous one"
  type        = number
  default     = 3

  validation {
    condition     = var.policy_password_reuse_prevention >= 0 && var.policy_password_reuse_prevention <= 24
    error_message = "The valid value of the password reuse prevention feature of the identity center's password policy is range from 0 to 24"
  }
}

variable "policy_password_not_username_or_invert" {
  description = "Whether the password can be the username or the username spelled backwards"
  type        = bool
  default     = false
}

variable "policy_password_validity_period" {
  description = "The password validity period (days)"
  type        = number
  default     = 7

  validation {
    condition     = var.policy_password_validity_period >= 0 && var.policy_password_validity_period <= 180
    error_message = "The valid value of the password validity period (days) is range from 0 to 180"
  }
}

variable "policy_password_char_combination" {
  description = "The minimum number of character types that a password must contain"
  type        = number

  validation {
    condition     = var.policy_password_char_combination >= 2 && var.policy_password_char_combination <= 4
    error_message = "The valid value of the minimum number of character types that a password must contain is range from 2 to 4"
  }
}

variable "policy_allow_user_to_change_password" {
  description = "Whether IAM users are allowed to change their own passwords"
  type        = bool
  default     = true
}

# Create IAM password policy resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identityv5_password_policy" "test" {
  maximum_consecutive_identical_chars = var.policy_max_consecutive_identical_chars
  minimum_password_age                = var.policy_min_password_age
  minimum_password_length             = var.policy_min_password_length
  password_reuse_prevention           = var.policy_password_reuse_prevention
  password_not_username_or_invert     = var.policy_password_not_username_or_invert
  password_validity_period            = var.policy_password_validity_period
  password_char_combination           = var.policy_password_char_combination
  allow_user_to_change_password       = var.policy_allow_user_to_change_password
}
```

**Parameter Description**:
- **maximum_consecutive_identical_chars**: Maximum number of times that a character is allowed to consecutively present in a password, assigned by referencing input variable policy_max_consecutive_identical_chars, valid range is 0-32
- **minimum_password_age**: Minimum password age (minutes), assigned by referencing input variable policy_min_password_age, valid range is 0-1440
- **minimum_password_length**: Minimum password length, assigned by referencing input variable policy_min_password_length, valid range is 8-32
- **password_reuse_prevention**: Password reuse prevention, assigned by referencing input variable policy_password_reuse_prevention, valid range is 0-24, default value is 3
- **password_not_username_or_invert**: Password cannot be username or username spelled backwards, assigned by referencing input variable policy_password_not_username_or_invert, default value is false
- **password_validity_period**: Password validity period (days), assigned by referencing input variable policy_password_validity_period, valid range is 0-180, default value is 7
- **password_char_combination**: Minimum number of character types that a password must contain, assigned by referencing input variable policy_password_char_combination, valid range is 2-4
- **allow_user_to_change_password**: Whether IAM users are allowed to change their own passwords, assigned by referencing input variable policy_allow_user_to_change_password, default value is true

> Note: IAM password policy is a global resource, and after configuration, it will apply to all IAM users. Password policy parameters all have value range restrictions. Please configure reasonably according to actual security requirements. It is recommended to follow the principle of least privilege and set reasonable password complexity requirements to improve account security.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# IAM Password Policy Configuration
policy_max_consecutive_identical_chars = 2
policy_min_password_age                = 60
policy_min_password_length             = 8
policy_password_char_combination       = 2
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially:
   - `policy_max_consecutive_identical_chars` can be set to the maximum number of times that a character is allowed to consecutively present in a password, recommended to set to 2 or smaller
   - `policy_min_password_age` can be set to the minimum password age (minutes), recommended to set to 60 minutes or longer
   - `policy_min_password_length` can be set to the minimum password length, recommended to set to 8 or longer
   - `policy_password_reuse_prevention` can be set to password reuse prevention, recommended to set to 3 or larger
   - `policy_password_not_username_or_invert` can be set to true to prohibit password from being username or username spelled backwards
   - `policy_password_validity_period` can be set to password validity period (days), recommended to set to 30-90 days
   - `policy_password_char_combination` can be set to the minimum number of character types that a password must contain, recommended to set to 2 or larger
   - `policy_allow_user_to_change_password` can be set to true to allow IAM users to change their own passwords
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="policy_min_password_length=12" -var="policy_password_char_combination=3"`
2. Environment variables: `export TF_VAR_policy_min_password_length=12` and `export TF_VAR_policy_password_char_combination=3`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values. After IAM password policy is configured, it will apply to all IAM users. Please configure password policy parameters reasonably according to actual security requirements.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create IAM password policy:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating IAM password policy
4. Run `terraform show` to view the details of the created IAM password policy

> Note: IAM password policy is a global resource, and after configuration, it will apply to all IAM users. Password policy parameters all have value range restrictions. Please ensure parameter values are within the valid range. It is recommended to understand the current password usage of IAM users before configuring password policy to avoid overly strict policies that prevent users from normal use.

## Reference Information

- [Huawei Cloud IAM Product Documentation](https://support.huaweicloud.com/iam/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Password Policy](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/iam/v5/password-policy)
