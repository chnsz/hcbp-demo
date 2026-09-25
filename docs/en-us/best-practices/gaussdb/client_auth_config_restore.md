# Deploy Client Access Authentication Configuration Restore

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud. It supports both centralized and distributed deployment modes and is widely used in core business scenarios such as finance, government, and e-commerce that demand high data consistency and reliability. GaussDB uses client access authentication (HBA) configuration to control the client addresses, authentication methods, and database users that are allowed to access the database, which is an important part of database security protection.

This best practice will introduce how to use Terraform to automatically restore the client access authentication configuration of a GaussDB instance, including specifying the instance, selecting a history record version, or restoring to the default configuration.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [GaussDB Client Access Authentication Configuration Restore (huaweicloud_gaussdb_client_auth_config_restore)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_client_auth_config_restore)

### Resource/Data Source Dependencies

```
huaweicloud_gaussdb_client_auth_config_restore
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Restore Client Access Authentication Configuration

Add the following script in the TF file (such as main.tf):

```hcl
# Restore the client access authentication configuration of a GaussDB instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_id" {
  description = "The ID of the GaussDB instance"
  type        = string
  default     = ""
}

variable "hba_history_id" {
  description = "The client access authentication modification history record ID"
  type        = string
  default     = ""
}

resource "huaweicloud_gaussdb_client_auth_config_restore" "test" {
  instance_id    = var.instance_id
  hba_history_id = var.hba_history_id
}
```

**Parameter Description**:

- **instance_id**: The ID of the GaussDB instance, assigned by referencing the input variable instance_id
- **hba_history_id**: The client access authentication modification history record ID, assigned by referencing the input variable hba_history_id; if empty, it means restoring to the default configuration

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication parameters
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Resource parameters
instance_id    = "your_instance_id"
hba_history_id = "your_hba_history_id"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="instance_id=my-instance-id"`
2. Environment variables: `export TF_VAR_instance_id=my-instance-id`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to restore the client access authentication configuration:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start restoring the client access authentication configuration
4. Run `terraform show` to view the restored client access authentication configuration

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Client Access Authentication Configuration Restore](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/client-auth-config-restore)
