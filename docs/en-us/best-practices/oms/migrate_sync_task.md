# Deploy Migration Synchronization Task

## Application Scenario

Object Storage Migration Service (OMS) is a one-stop data migration service provided by Huawei Cloud, supporting the migration of object data from other cloud service providers or local environments to Huawei Cloud Object Storage Service (OBS). In cross-cloud and cross-region data migration scenarios, besides one-time migration, it is often necessary to continuously keep the source and destination data consistent.

This best practice will introduce how to use Terraform to automatically deploy an OMS migration synchronization task, including creating source and destination OBS buckets, and creating a migration synchronization task with configurations such as the source cloud service provider, source region, consistency check method, and metadata migration, helping you quickly build a sustainable object data migration capability.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Source OBS Bucket (huaweicloud_obs_bucket)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [Destination OBS Bucket (huaweicloud_obs_bucket)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [Migration Synchronization Task (huaweicloud_oms_migration_sync_task)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/oms_migration_sync_task)

### Resource/Data Source Dependencies

```
huaweicloud_obs_bucket.source
    └── huaweicloud_oms_migration_sync_task.test

huaweicloud_obs_bucket.dest
    └── huaweicloud_oms_migration_sync_task.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the article [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Source OBS Bucket

Add the following script in the TF file (such as main.tf) to create a source OBS bucket:

```hcl
# Create a source OBS bucket resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "source_bucket_name" {
  description = "The name of the source OBS bucket"
  type        = string
}

variable "bucket_storage_class" {
  description = "The storage class of the OBS bucket"
  type        = string
  default     = "STANDARD"
}

variable "bucket_acl" {
  description = "The ACL of the OBS bucket"
  type        = string
  default     = "private"
}

variable "bucket_force_destroy" {
  description = "Whether to force destroy the OBS bucket"
  type        = bool
  default     = true
}

resource "huaweicloud_obs_bucket" "source" {
  bucket        = var.source_bucket_name
  storage_class = var.bucket_storage_class
  acl           = var.bucket_acl
  force_destroy = var.bucket_force_destroy
}
```

**Parameter Description**:
- **bucket**: The bucket name, assigned by referencing the input variable source_bucket_name
- **storage_class**: The storage class of the bucket, assigned by referencing the input variable bucket_storage_class, defaulting to STANDARD
- **acl**: The access control policy of the bucket, assigned by referencing the input variable bucket_acl, defaulting to private
- **force_destroy**: Whether to force destroy the bucket, assigned by referencing the input variable bucket_force_destroy, defaulting to true

### 3. Create a Destination OBS Bucket

Add the following script in the TF file (such as main.tf) to create a destination OBS bucket:

```hcl
# Create a destination OBS bucket resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dest_bucket_name" {
  description = "The name of the destination OBS bucket"
  type        = string
}

resource "huaweicloud_obs_bucket" "dest" {
  bucket        = var.dest_bucket_name
  storage_class = var.bucket_storage_class
  acl           = var.bucket_acl
  force_destroy = var.bucket_force_destroy
}
```

**Parameter Description**:
- **bucket**: The bucket name, assigned by referencing the input variable dest_bucket_name
- **storage_class**: The storage class of the bucket, assigned by referencing the input variable bucket_storage_class
- **acl**: The access control policy of the bucket, assigned by referencing the input variable bucket_acl
- **force_destroy**: Whether to force destroy the bucket, assigned by referencing the input variable bucket_force_destroy

### 4. Create a Migration Synchronization Task

Add the following script in the TF file (such as main.tf) to create a migration synchronization task:

```hcl
# Create a migration synchronization task resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "source_cloud_type" {
  description = "The source cloud service provider"
  type        = string
  default     = "HuaweiCloud"
}

variable "source_region" {
  description = "The region where the source bucket is located"
  type        = string
}

variable "source_access_key" {
  description = "The access key for accessing the source bucket"
  type        = string
  sensitive   = true
}

variable "source_secret_key" {
  description = "The secret key for accessing the source bucket"
  type        = string
  sensitive   = true
}

variable "dest_access_key" {
  description = "The access key for accessing the destination bucket"
  type        = string
  sensitive   = true
}

variable "dest_secret_key" {
  description = "The secret key for accessing the destination bucket"
  type        = string
  sensitive   = true
}

variable "task_description" {
  description = "The description of the migration synchronization task"
  type        = string
  default     = ""
}

variable "consistency_check" {
  description = "The consistency check method"
  type        = string
  default     = "size_last_modified"
}

variable "enable_metadata_migration" {
  description = "Whether to enable metadata migration"
  type        = bool
  default     = false
}

resource "huaweicloud_oms_migration_sync_task" "test" {
  src_cloud_type            = var.source_cloud_type
  src_region                = var.source_region
  src_ak                    = var.source_access_key
  src_sk                    = var.source_secret_key
  src_bucket                = huaweicloud_obs_bucket.source.bucket
  dst_ak                    = var.dest_access_key
  dst_sk                    = var.dest_secret_key
  dst_bucket                = huaweicloud_obs_bucket.dest.bucket
  description               = var.task_description
  consistency_check         = var.consistency_check
  enable_metadata_migration = var.enable_metadata_migration
}
```

**Parameter Description**:
- **src_cloud_type**: The source cloud service provider, assigned by referencing the input variable source_cloud_type, defaulting to HuaweiCloud
- **src_region**: The region where the source bucket is located, assigned by referencing the input variable source_region
- **src_ak**: The access key (AK) for accessing the source bucket, assigned by referencing the input variable source_access_key
- **src_sk**: The secret key (SK) for accessing the source bucket, assigned by referencing the input variable source_secret_key
- **src_bucket**: The source bucket name, assigned by referencing the bucket attribute of the source OBS bucket resource
- **dst_ak**: The access key (AK) for accessing the destination bucket, assigned by referencing the input variable dest_access_key
- **dst_sk**: The secret key (SK) for accessing the destination bucket, assigned by referencing the input variable dest_secret_key
- **dst_bucket**: The destination bucket name, assigned by referencing the bucket attribute of the destination OBS bucket resource
- **description**: The description of the migration synchronization task, assigned by referencing the input variable task_description
- **consistency_check**: The consistency check method, assigned by referencing the input variable consistency_check, defaulting to size_last_modified
- **enable_metadata_migration**: Whether to enable metadata migration, assigned by referencing the input variable enable_metadata_migration, defaulting to false

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "YourAccessKey"
secret_key  = "YourSecretKey"

# Resource variables
source_bucket_name = "tf-test-source"
dest_bucket_name   = "tf-test-dest"
source_region      = "cn-north-4"
source_access_key  = "YourSourceBucketAccessKey"
source_secret_key  = "YourSourceBucketSecretKey"
dest_access_key    = "YourDestBucketAccessKey"
dest_secret_key    = "YourDestBucketSecretKey"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="source_bucket_name=my-source-bucket"`
2. Environment variables: `export TF_VAR_source_bucket_name=my-source-bucket`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the migration synchronization task
4. Run `terraform show` to view the created migration synchronization task

## Reference Information

- [Huawei Cloud OMS Product Documentation](https://support.huaweicloud.com/oms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For OMS Migration Synchronization Task](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/oms/migrate-sync-task)
