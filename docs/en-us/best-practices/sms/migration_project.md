# Deploy Migration Project

## Application Scenario

Server Migration Service (SMS) is a server migration service provided by Huawei Cloud, supporting migration of physical servers, virtual machines, or servers from other cloud platforms to Huawei Cloud, achieving seamless business migration. Migration projects are fundamental resources of SMS service, used to manage and organize migration tasks. By creating migration projects, parameters such as migration region, network type, server type, and synchronization policy can be configured, providing basic configuration for subsequent migration tasks. This best practice introduces how to use Terraform to automatically deploy migration projects, including project basic information, region configuration, network configuration, and synchronization policy configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Migration Project Resource (huaweicloud_sms_migration_project)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sms_migration_project)

### Resource/Data Source Dependencies

```text
huaweicloud_sms_migration_project
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create Migration Project

Add the following script to the TF file (such as main.tf) to create a migration project:

```hcl
variable "migration_project_name" {
  description = "The migration project name"
  type        = string
}

variable "migration_project_region" {
  description = "The region name"
  type        = string
}

variable "migration_project_use_public_ip" {
  description = "Whether to use a public IP address for migration"
  type        = bool
}

variable "migration_project_exist_server" {
  description = "Whether the server already exists"
  type        = bool
}

variable "migration_project_type" {
  description = "The migration project typ"
  type        = string
}

variable "migration_project_syncing" {
  description = "Whether to continue syncing after the first copy or sync"
  type        = bool
}

# Create migration project resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_sms_migration_project" "test" {
  name          = var.migration_project_name
  region        = var.migration_project_region
  use_public_ip = var.migration_project_use_public_ip
  exist_server  = var.migration_project_exist_server
  type          = var.migration_project_type
  syncing       = var.migration_project_syncing
}
```

**Parameter Description**:
- **name**: Migration project name, assigned by referencing the input variable `migration_project_name`
- **region**: Region name, assigned by referencing the input variable `migration_project_region`, used to specify the migration target region
- **use_public_ip**: Whether to use public IP for migration, assigned by referencing the input variable `migration_project_use_public_ip`
- **exist_server**: Whether the server already exists, assigned by referencing the input variable `migration_project_exist_server`
- **type**: Migration project type, assigned by referencing the input variable `migration_project_type`
- **syncing**: Whether to continue syncing after the first copy or sync, assigned by referencing the input variable `migration_project_syncing`

> Note: Migration projects are used to manage and organize migration tasks, and need to configure corresponding parameters according to actual migration scenarios. If using public IP for migration, ensure that the source server can access the public network.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Migration project basic information (Required)
migration_project_name          = "tf_test_sms_migration_project"
migration_project_region        = "cn-north-4"
migration_project_use_public_ip = true
migration_project_exist_server  = true
migration_project_type          = "LINUX"
migration_project_syncing       = true
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="migration_project_name=tf_test_sms_migration_project" -var="migration_project_region=cn-north-4"`
2. Environment variables: `export TF_VAR_migration_project_name=tf_test_sms_migration_project`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the migration project
4. Run `terraform show` to view the created migration project

## Reference Information

- [Huawei Cloud SMS Product Documentation](https://support.huaweicloud.com/sms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Migration Project](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sms/migration-project)
