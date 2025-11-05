# Deploy System Tracker

## Application Scenario

The System Tracker of Cloud Trace Service (CTS) is a core service used to record and store audit logs of cloud resource operations. Through system trackers, you can achieve security audit, compliance monitoring, operation recording, problem troubleshooting, and other functions.

System trackers are particularly suitable for scenarios that require recording cloud resource operation history, conducting security audits, meeting compliance requirements, etc., such as enterprise-level security monitoring, compliance checks, operation auditing, problem tracking, etc. This best practice will introduce how to use Terraform to automatically deploy a CTS system tracker, implementing automatic recording and storage of cloud resource operations.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

This best practice does not use data sources.

### Resources

- [OBS Bucket Resource (huaweicloud_obs_bucket)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [CTS System Tracker Resource (huaweicloud_cts_tracker)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cts_tracker)

### Resource/Data Source Dependencies

```
huaweicloud_obs_bucket
    └── huaweicloud_cts_tracker
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create OBS Bucket

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an OBS bucket resource:

```hcl
# Variable definitions for OBS resources
variable "bucket_name" {
  description = "The name of the OBS bucket for storing trace files"
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
- **acl**: Bucket access control list, set to "private" for private access
- **force_destroy**: Whether to force delete the bucket, set to true to allow deletion of non-empty buckets

### 3. Create CTS System Tracker

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CTS system tracker resource:

```hcl
# Variable definitions for CTS resources
variable "tracker_enabled" {
  description = "Whether to enable the system tracker"
  type        = bool
  default     = true
}

variable "tracker_tags" {
  description = "The tags of the system tracker"
  type        = map(string)
  default     = {}
}

variable "is_system_tracker_delete" {
  description = "Whether to delete the system tracker when the tracker resource is deleted"
  type        = bool
  default     = true
}

variable "trace_object_prefix" {
  description = "The prefix of the trace object in the OBS bucket"
  type        = string
}

variable "trace_file_compression_type" {
  description = "The compression type of the trace file"
  type        = string
  default     = "gzip"
}

variable "is_lts_enabled" {
  description = "Whether to enable the trace analysis for LTS service"
  type        = bool
  default     = true
}

# Create a CTS system tracker resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_cts_tracker" "test" {
  enabled        = var.tracker_enabled
  tags           = var.tracker_tags
  delete_tracker = var.is_system_tracker_delete
  bucket_name    = huaweicloud_obs_bucket.test.bucket
  file_prefix    = var.trace_object_prefix
  compress_type  = var.trace_file_compression_type
  lts_enabled    = var.is_lts_enabled
}
```

**Parameter Description**:
- **enabled**: Whether to enable the system tracker, assigned by referencing the input variable tracker_enabled, default value true means enabled
- **tags**: System tracker tags, assigned by referencing the input variable tracker_tags, used for resource classification and management
- **delete_tracker**: Whether to delete the system tracker when the tracker resource is deleted, assigned by referencing the input variable is_system_tracker_delete, default value true means delete
- **bucket_name**: OBS bucket name for storing trace data, assigned by referencing the OBS bucket's bucket attribute
- **file_prefix**: Prefix of trace objects in the OBS bucket, assigned by referencing the input variable trace_object_prefix
- **compress_type**: Compression type of trace files, assigned by referencing the input variable trace_file_compression_type, default value "gzip" means using gzip compression
- **lts_enabled**: Whether to enable trace analysis for LTS service, assigned by referencing the input variable is_lts_enabled, default value true means enabled

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# OBS bucket configuration
bucket_name = "tf-test-trace-bucket"

# CTS system tracker configuration
trace_object_prefix = "tf_test"
tracker_tags        = {
  "owner" = "terraform"
}
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="bucket_name=my-bucket" -var="trace_object_prefix=my-prefix"`
2. Environment variables: `export TF_VAR_bucket_name=my-bucket`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating OBS bucket and CTS system tracker
4. Run `terraform show` to view the details of the created OBS bucket and CTS system tracker

## Reference Information

- [Huawei Cloud CTS Product Documentation](https://support.huaweicloud.com/cts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CTS System Tracker](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cts/system-tracker)
