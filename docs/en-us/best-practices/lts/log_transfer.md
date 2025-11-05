# Deploy Log Transfer

## Application Scenario

Huawei Cloud Log Tank Service (LTS) log transfer functionality allows users to periodically transfer log data from LTS log groups and log streams to OBS (Object Storage Service), implementing long-term storage and backup of log data. By configuring log transfer tasks, you can implement automated log data archiving, cost optimization, and compliance requirements.

This best practice is particularly suitable for scenarios that require long-term log data preservation, implementing log data backup, meeting compliance audit requirements, and optimizing storage costs, such as log archiving, data lake construction, compliance auditing, etc. This best practice will introduce how to use Terraform to automatically deploy LTS log transfer, including log group, log stream, OBS bucket, and log transfer task creation, implementing a complete log transfer solution.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Log Group Resource (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [Log Stream Resource (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [OBS Bucket Resource (huaweicloud_obs_bucket)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [Log Transfer Resource (huaweicloud_lts_transfer)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_transfer)

### Resource/Data Source Dependencies

```
huaweicloud_lts_group
    ├── huaweicloud_lts_stream
    └── huaweicloud_lts_transfer

huaweicloud_obs_bucket
    └── huaweicloud_lts_transfer
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Log Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a log group resource:

```hcl
variable "group_name" {
  description = "The name of the log group"
  type        = string
}

variable "group_log_expiration_days" {
  description = "The log expiration days of the log group"
  type        = number
  default     = 14
}

# Create a log group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_lts_group" "test" {
  group_name  = var.group_name
  ttl_in_days = var.group_log_expiration_days
}
```

**Parameter Description**:
- **group_name**: Log group name, assigned by referencing the input variable group_name
- **ttl_in_days**: Log expiration days for the log group, assigned by referencing the input variable group_log_expiration_days, default value is 14

### 3. Create Log Stream

Add the following script to the TF file to instruct Terraform to create a log stream resource:

```hcl
variable "stream_name" {
  description = "The name of the log stream"
  type        = string
}

variable "stream_log_expiration_days" {
  description = "The log expiration days of the log stream"
  type        = number
  default     = null
}

# Create a log stream resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.stream_name
  ttl_in_days = var.stream_log_expiration_days
}
```

**Parameter Description**:
- **group_id**: Log group ID that the log stream belongs to, referencing the ID of the previously created log group resource
- **stream_name**: Log stream name, assigned by referencing the input variable stream_name
- **ttl_in_days**: Log expiration days for the log stream, assigned by referencing the input variable stream_log_expiration_days, default value is null (inherits log group settings)

### 4. Create OBS Bucket

Add the following script to the TF file to instruct Terraform to create an OBS bucket resource:

```hcl
variable "bucket_name" {
  description = "The name of the OBS bucket"
  type        = string
}

# Create an OBS bucket resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_obs_bucket" "test" {
  bucket        = var.bucket_name
  acl           = "private"
  force_destroy = true
}
```

**Parameter Description**:
- **bucket**: OBS bucket name, assigned by referencing the input variable bucket_name
- **acl**: Bucket ACL policy, set to "private" for private access
- **force_destroy**: Whether to force delete the bucket, set to true to allow deletion of non-empty buckets

### 5. Create Log Transfer Task

Add the following script to the TF file to instruct Terraform to create a log transfer resource:

```hcl
variable "transfer_type" {
  description = "The type of the log transfer"
  type        = string
  default     = "OBS"
}

variable "transfer_mode" {
  description = "The mode of the log transfer"
  type        = string
  default     = "cycle"
}

variable "transfer_storage_format" {
  description = "The storage format of the log transfer"
  type        = string
  default     = "JSON"
}

variable "transfer_status" {
  description = "The status of the log transfer"
  type        = string
  default     = "ENABLE"
}

variable "bucket_dir_prefix_name" {
  description = "The prefix path of the OBS transfer task"
  type        = string
  default     = "LTS-test/%GroupName/%StreamName/%Y/%m/%d/%H/%M"
}

variable "bucket_time_zone" {
  description = "The time zone of the OBS bucket"
  type        = string
  default     = "UTC"
}

variable "bucket_time_zone_id" {
  description = "The time zone ID of the OBS bucket"
  type        = string
  default     = "Etc/GMT"
}

# Create a log transfer resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_lts_transfer" "test" {
  log_group_id = huaweicloud_lts_group.test.id

  log_streams {
    log_stream_id = huaweicloud_lts_stream.test.id
  }

  log_transfer_info {
    log_transfer_type   = var.transfer_type
    log_transfer_mode   = var.transfer_mode
    log_storage_format  = var.transfer_storage_format
    log_transfer_status = var.transfer_status

    log_transfer_detail {
      obs_bucket_name     = huaweicloud_obs_bucket.test.bucket
      obs_period          = 3
      obs_period_unit     = "hour"
      obs_dir_prefix_name = var.bucket_dir_prefix_name
      obs_time_zone       = var.bucket_time_zone
      obs_time_zone_id    = var.bucket_time_zone_id
    }
  }

  depends_on = [
    huaweicloud_obs_bucket.test,
  ]
}
```

**Parameter Description**:
- **log_group_id**: Log group ID that the log transfer belongs to, referencing the ID of the previously created log group resource
- **log_streams**: Log stream configuration block
  - **log_stream_id**: Log stream ID to transfer, referencing the ID of the previously created log stream resource
- **log_transfer_info**: Log transfer information configuration block
  - **log_transfer_type**: Log transfer type, assigned by referencing the input variable transfer_type, default value is "OBS"
  - **log_transfer_mode**: Log transfer mode, assigned by referencing the input variable transfer_mode, default value is "cycle" (periodic transfer)
  - **log_storage_format**: Log storage format, assigned by referencing the input variable transfer_storage_format, default value is "JSON"
  - **log_transfer_status**: Log transfer status, assigned by referencing the input variable transfer_status, default value is "ENABLE"
- **log_transfer_detail**: Log transfer detail configuration block
  - **obs_bucket_name**: OBS bucket name, referencing the name of the previously created OBS bucket resource
  - **obs_period**: Transfer period, set to 3 to transfer every 3 time units
  - **obs_period_unit**: Transfer period unit, set to "hour" for hourly calculation
  - **obs_dir_prefix_name**: OBS directory prefix name, assigned by referencing the input variable bucket_dir_prefix_name, supports variable substitution
  - **obs_time_zone**: OBS time zone, assigned by referencing the input variable bucket_time_zone, default value is "UTC"
  - **obs_time_zone_id**: OBS time zone ID, assigned by referencing the input variable bucket_time_zone_id, default value is "Etc/GMT"
- **depends_on**: Explicit dependency relationship, ensuring the OBS bucket exists before log transfer task creation

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Log group configuration
group_name = "tf_test_log_group"

# Log stream configuration
stream_name = "tf_test_log_stream"

# OBS bucket configuration
bucket_name = "tf-test-log-transfer-obs-bucket"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="group_name=my-group" -var="stream_name=my-stream"`
2. Environment variables: `export TF_VAR_group_name=my-group`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating log transfer
4. Run `terraform show` to view the created log transfer details

## Reference Information

- [Huawei Cloud Log Tank Service Product Documentation](https://support.huaweicloud.com/lts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For LTS Log Transfer](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/lts/log-transfer)
