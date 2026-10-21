# Deploy Backup Download Authorization

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud. When users need to download backup files of a GaussDB instance to local storage for offline analysis, archiving, or migration, they must first authorize the backup download to obtain the OBS bucket name where the backup files are stored and the backup file paths that can be downloaded using OBS Browser+.

This best practice will introduce how to use Terraform to automatically deploy GaussDB backup download authorization, including specifying the backup ID and creating the authorization resource, helping users quickly obtain the download information of backup files.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [GaussDB Backup Download Authorization (huaweicloud_gaussdb_download_backups_authorize)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_download_backups_authorize)

### Resource/Data Source Dependencies

```
huaweicloud_gaussdb_download_backups_authorize
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create GaussDB Backup Download Authorization

Add the following script to the TF file (such as main.tf):

```hcl
# Create the GaussDB backup download authorization resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "backup_id" {
  description = "The backup ID in UUID format (36 characters) that uniquely identifies a GaussDB backup"
  type        = string
}

resource "huaweicloud_gaussdb_download_backups_authorize" "test" {
  backup_id = var.backup_id
}
```

**Parameter Description**:
- **backup_id**: The GaussDB backup ID, assigned by referencing the input variable backup_id, in 36-character UUID format, used to uniquely identify the backup to be authorized for download

> Note: This resource is a one-time action resource. The create operation sends a POST request to the GaussDB (opengauss) service endpoint to complete the backup download authorization. The resource ID equals backup_id. The backup_id parameter is non-updatable and will trigger resource replacement if changed. The read and update operations are no-ops. Deleting this resource only removes it from the Terraform state with a warning, and the authorization is not revoked in the cloud. After successful creation, the computed attributes `bucket` and `file_paths` can be used to obtain the OBS bucket name where the backup file is stored and the backup file paths that can be downloaded using OBS Browser+.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
backup_id = "your_gaussdb_backup_id"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="backup_id=your_gaussdb_backup_id"`
2. Environment variables: `export TF_VAR_backup_id=your_gaussdb_backup_id`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB backup download authorization
4. Run `terraform show` to view the created GaussDB backup download authorization

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Backup Download Authorization](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-download-backups-authorize)
