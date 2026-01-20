# Deploy Account

## Application Scenario

Huawei Cloud Organizations service is a multi-account management service provided by Huawei Cloud, supporting enterprises to centrally manage multiple Huawei Cloud accounts through organizational structure, achieving centralized resource management and unified permission control. Account is a core resource in Organizations service, used to create and manage sub-accounts under organizations or organizational units. By creating accounts, resource isolation, cost allocation, and permission management can be achieved in multi-account architectures, meeting enterprise-level multi-account management needs. This best practice introduces how to use Terraform to automatically deploy Organizations accounts, including account creation and configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Organizations Account Resource (huaweicloud_organizations_account)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/organizations_account)

### Resource/Data Source Dependencies

```text
huaweicloud_organizations_account
```

> Note: Organizations account is an independent resource that does not depend on other resources. Accounts can be created under root organizations or organizational units, specified through the parent_id parameter.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create Organizations Account

Add the following script to the TF file (such as main.tf) to create an Organizations account:

```hcl
resource "huaweicloud_organizations_account" "test" {
  name        = var.name
  email       = var.email
  phone       = var.phone
  agency_name = var.agency_name
  parent_id   = var.parent_id
  description = var.description
  tags        = var.tags
}
```

**Parameter Description**:
- **name**: Account name, assigned by referencing the input variable `name`, used to identify the account
- **email**: Account email address, assigned by referencing the input variable `email`, used for account login and notifications
- **phone**: Account mobile number, assigned by referencing the input variable `phone`, used for account verification and notifications
- **agency_name**: Agency name, assigned by referencing the input variable `agency_name`, used to specify the account's agency relationship
- **parent_id**: Parent organization or organizational unit ID, assigned by referencing the input variable `parent_id`, used to specify the organization or organizational unit to which the account belongs
- **description**: Account description, assigned by referencing the input variable `description`, used to describe the account's purpose and related information
- **tags**: Account tags, assigned by referencing the input variable `tags`, used to classify and manage accounts

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Organizations account configuration (Required)
name        = "tf_test_account"
email       = "test@example.com"
phone       = "tf_test_phone"
agency_name = "tf_test_agency_name"
parent_id   = "tf_test_parent_id"
description = "Created by terraform script"

tags = {
  key   = "value"
  owner = "terraform"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="name=tf_test_account"`
2. Environment variables: `export TF_VAR_name=tf_test_account`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating Organizations account
4. Run `terraform show` to view the created Organizations account

## Reference Information

- [Huawei Cloud Organizations Product Documentation](https://support.huaweicloud.com/intl/en-us/organizations/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Account](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/organizations/organization-account)
