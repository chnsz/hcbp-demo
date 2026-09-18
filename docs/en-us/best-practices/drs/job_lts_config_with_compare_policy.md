# Deploy Job LTS Configuration and Compare Policy

## Application Scenario

Data Replication Service (DRS) is a one-stop data replication service provided by Huawei Cloud, supporting scenarios such as database cloud migration, database migration, real-time database synchronization, and database disaster recovery. During the running of a real-time synchronization or migration job, users often need to deliver job logs to Log Tank Service (LTS) for centralized retrieval and analysis, and to detect data differences between the source and destination through a periodic data comparison policy to ensure data consistency.

This best practice will introduce how to use Terraform to configure LTS log delivery for an existing DRS job and enable a periodic data comparison policy. The practice first creates an LTS log group and stream, then enables the LTS log delivery of the DRS job, and finally opens a periodic data comparison policy for the same job, helping you efficiently manage the log and data verification capabilities of DRS jobs using Infrastructure as Code (IaC).

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [LTS Log Group (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS Log Stream (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [DRS Job LTS Configuration (huaweicloud_drs_lts_config)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_lts_config)
- [DRS Job Compare Policy (huaweicloud_drs_compare_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_compare_policy)

### Resource/Data Source Dependencies

```
huaweicloud_lts_group
    └── huaweicloud_lts_stream
            └── huaweicloud_drs_lts_config
                    └── huaweicloud_drs_compare_policy
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create an LTS Log Group

Add the following script to the TF file (such as main.tf) to create an LTS log group for storing DRS job logs:

```hcl
# Create an LTS log group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "lts_group_name" {
  description = "The name of the LTS group used to store the DRS job logs"
  type        = string
}

resource "huaweicloud_lts_group" "test" {
  group_name  = var.lts_group_name
  ttl_in_days = 30
}
```

**Parameter Description**:
- **group_name**: The name of the log group, assigned by referencing the input variable lts_group_name
- **ttl_in_days**: The log storage duration in days, set to 30 here

### 3. Create an LTS Log Stream

Add the following script to the TF file (such as main.tf) to create a log stream under the log group:

```hcl
# Create an LTS log stream resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "lts_stream_name" {
  description = "The name of the LTS stream used to store the DRS job logs"
  type        = string
}

resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.lts_stream_name
}
```

**Parameter Description**:
- **group_id**: The ID of the log group to which the log stream belongs, assigned by referencing the ID of the log group resource huaweicloud_lts_group.test created in the previous step
- **stream_name**: The name of the log stream, assigned by referencing the input variable lts_stream_name

### 4. Configure LTS Log Delivery for the DRS Job

Add the following script to the TF file (such as main.tf) to enable the LTS log delivery of the DRS job:

```hcl
# Create a DRS job LTS configuration resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "drs_job_id" {
  description = "The ID of the existing DRS job"
  type        = string
}

resource "huaweicloud_drs_lts_config" "test" {
  job_id        = var.drs_job_id
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id
}
```

**Parameter Description**:
- **job_id**: The ID of the existing DRS job, assigned by referencing the input variable drs_job_id
- **log_group_id**: The ID of the log group, assigned by referencing the ID of the log group resource huaweicloud_lts_group.test created earlier
- **log_stream_id**: The ID of the log stream, assigned by referencing the ID of the log stream resource huaweicloud_lts_stream.test created earlier

### 5. Configure the Data Compare Policy for the DRS Job

Add the following script to the TF file (such as main.tf) to enable the periodic data compare policy of the DRS job:

```hcl
# Create a DRS job compare policy resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "compare_policy_period" {
  description = "The comparison period of the compare policy, e.g. * * 1,3,5 for weekly comparison"
  type        = string
}

variable "compare_policy_begin_time" {
  description = "The start time when the comparison policy takes effect, UTC time in HH:mm:ss format"
  type        = string
}

variable "compare_policy_end_time" {
  description = "The end time when the comparison policy takes effect, UTC time in HH:mm:ss format"
  type        = string
}

variable "compare_policy_compare_type" {
  description = "The list of comparison types, valid values are object_comparison, lines and account"
  type        = list(string)
  default     = ["lines"]
}

variable "compare_policy_compare_policy" {
  description = "The comparison policy, valid values are normal and manyToOne"
  type        = string
  default     = "normal"
}

variable "compare_policy_interval_hour" {
  description = "The comparison interval in hours, required for hourly comparison"
  type        = number
  default     = null
}

resource "huaweicloud_drs_compare_policy" "test" {
  job_id         = var.drs_job_id
  period         = var.compare_policy_period
  begin_time     = var.compare_policy_begin_time
  end_time       = var.compare_policy_end_time
  compare_type   = var.compare_policy_compare_type
  compare_policy = var.compare_policy_compare_policy
  interval_hour  = var.compare_policy_interval_hour

  depends_on = [huaweicloud_drs_lts_config.test]
}
```

**Parameter Description**:
- **job_id**: The ID of the existing DRS job, assigned by referencing the input variable drs_job_id
- **period**: The comparison period, assigned by referencing the input variable compare_policy_period, for example `* * 1,3,5` means comparison is performed every Monday, Wednesday, and Friday
- **begin_time**: The start time when the comparison policy takes effect (UTC time in HH:mm:ss format), assigned by referencing the input variable compare_policy_begin_time
- **end_time**: The end time when the comparison policy takes effect (UTC time in HH:mm:ss format), assigned by referencing the input variable compare_policy_end_time
- **compare_type**: The list of comparison types, assigned by referencing the input variable compare_policy_compare_type, valid values are object_comparison, lines and account
- **compare_policy**: The comparison policy, assigned by referencing the input variable compare_policy_compare_policy, valid values are normal and manyToOne
- **interval_hour**: The comparison interval in hours for hourly comparison, assigned by referencing the input variable compare_policy_interval_hour
- **depends_on**: Explicitly declares the dependency on the DRS job LTS configuration resource, ensuring that the LTS log delivery is configured before the compare policy is enabled

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Resource variables
lts_group_name            = "tf-test-drs-lts-group"
lts_stream_name           = "tf-test-drs-lts-stream"
drs_job_id                = "your-drs-job-id"
compare_policy_period     = "* * 1,3,5"
compare_policy_begin_time = "00:00:00"
compare_policy_end_time   = "04:00:00"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="drs_job_id=your-drs-job-id"`
2. Environment variables: `export TF_VAR_drs_job_id=your-drs-job-id`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to configure the LTS log delivery and data compare policy of the DRS job
4. Run `terraform show` to view the configured LTS log delivery and data compare policy of the DRS job

## Reference Information

- [Huawei Cloud Data Replication Service Product Documentation](https://support.huaweicloud.com/drs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DRS Job LTS Configuration and Compare Policy](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/job-lts-config-with-compare-policy)
