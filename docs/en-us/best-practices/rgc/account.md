# Deploy Account

## Application Scenario

Resource Governance Center (RGC) is a resource governance service provided by Huawei Cloud, supporting multi-account management, organizational unit management, blueprint configuration, and other functions to help you uniformly manage and govern cloud resources. By creating RGC accounts, new accounts can be created within organizational units, achieving unified account management and governance. This best practice introduces how to use Terraform to automatically deploy RGC accounts, including account basic information, identity store information, and organizational unit information configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Account Resource (huaweicloud_rgc_account)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rgc_account)

### Resource/Data Source Dependencies

```text
huaweicloud_rgc_account
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create Account

Add the following script to the TF file (such as main.tf) to create an RGC account:

```hcl
variable "account_name" {
  description = "The name of RGC account"
  type        = string
}

variable "account_email" {
  description = "The email of RGC account"
  type        = string
}

variable "account_phone" {
  description = "The phone number of RGC account"
  type        = string
  sensitive   = true
  default     = ""
}

variable "identity_store_user_name" {
  description = "The identity store user name of RGC account"
  type        = string
}

variable "identity_store_email" {
  description = "The identity store email of RGC account"
  type        = string
}

variable "parent_organizational_unit_name" {
  description = "The parent organizational unit name of RGC account"
  type        = string
}

variable "parent_organizational_unit_id" {
  description = "The parent organizational unit ID of RGC account"
  type        = string
}

# Create account resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_rgc_account" "test" {
  name                            = var.account_name
  email                           = var.account_email
  phone                           = var.account_phone
  identity_store_user_name        = var.identity_store_user_name
  identity_store_email            = var.identity_store_email
  parent_organizational_unit_name = var.parent_organizational_unit_name
  parent_organizational_unit_id   = var.parent_organizational_unit_id
}
```

**Parameter Description**:
- **name**: Account name, assigned by referencing the input variable `account_name`
- **email**: Account email, assigned by referencing the input variable `account_email`
- **phone**: Account phone number, assigned by referencing the input variable `account_phone`, optional parameter
- **identity_store_user_name**: Identity store user name, assigned by referencing the input variable `identity_store_user_name`
- **identity_store_email**: Identity store email, assigned by referencing the input variable `identity_store_email`
- **parent_organizational_unit_name**: Parent organizational unit name, assigned by referencing the input variable `parent_organizational_unit_name`
- **parent_organizational_unit_id**: Parent organizational unit ID, assigned by referencing the input variable `parent_organizational_unit_id`

> Note: Accounts must belong to an organizational unit, and the parent organizational unit name and ID must be specified. Identity store information is used for account identity authentication and access control.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Account basic information (Required)
account_name  = "tf-test-account"
account_email = "tf-test-account@terraform.com"
account_phone = "13123456789"

# Identity store information (Required)
identity_store_user_name = "tf-test-account"
identity_store_email     = "tf-test-account@terraform.com"

# Organizational unit information (Required)
parent_organizational_unit_name = "your-org-unit-name"
parent_organizational_unit_id   = "your-org-unit-id"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="account_name=tf-test-account" -var="account_email=tf-test-account@terraform.com"`
2. Environment variables: `export TF_VAR_account_name=tf-test-account`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the account
4. Run `terraform show` to view the created account

## Reference Information

- [Huawei Cloud RGC Product Documentation](https://support.huaweicloud.com/rgc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Account](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rgc/account)
