# Deploy Redis Diagnosis Task

## Application Scenario

During long-term operation, DCS Redis instances may develop potential issues such as big keys, slow queries, abnormal connection counts, and high memory fragmentation, which affect cache performance and stability. A diagnosis task intelligently analyzes the running status of a Redis instance within a specified time range, and outputs statistics on abnormal and failed items as well as per-node diagnosis reports, helping O&M personnel quickly locate and handle hidden risks.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis diagnosis task, which diagnoses an existing Redis instance within a specified time range and obtains the diagnosis results.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DCS Redis Diagnosis Task (huaweicloud_dcs_diagnosis_task)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_diagnosis_task)

### Resource/Data Source Dependencies

```
huaweicloud_dcs_diagnosis_task
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a DCS Redis Diagnosis Task

Add the following script in the TF file (such as main.tf) to create a DCS Redis diagnosis task:

```hcl
# Create a DCS Redis diagnosis task resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_id" {
  description = "The ID of the DCS instance to diagnose"
  type        = string
}

variable "begin_time" {
  description = "The start time of the diagnosis task, in RFC3339 format"
  type        = string
}

variable "end_time" {
  description = "The end time of the diagnosis task, in RFC3339 format"
  type        = string
}

resource "huaweicloud_dcs_diagnosis_task" "test" {
  instance_id = var.instance_id
  begin_time  = var.begin_time
  end_time    = var.end_time
}
```

**Parameter Description**:

- **instance_id**: The ID of the DCS instance to diagnose, assigned by referencing the input variable instance_id
- **begin_time**: The start time of the diagnosis task, in RFC3339 format, assigned by referencing the input variable begin_time
- **end_time**: The end time of the diagnosis task, in RFC3339 format, assigned by referencing the input variable end_time

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource information
instance_id = "your_dcs_instance_id"
begin_time  = "2024-01-01T00:00:00Z"
end_time    = "2024-01-02T00:00:00Z"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="instance_id=your_dcs_instance_id"`
2. Environment variables: `export TF_VAR_instance_id=your_dcs_instance_id`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DCS Redis diagnosis task
4. Run `terraform show` to view the created DCS Redis diagnosis task

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Diagnosis Task](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-diagnosis-task)
