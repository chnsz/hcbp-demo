# Deploy Redis Background Task Deletion

## Application Scenario

Distributed Cache Service (DCS) is a high-performance, highly available in-memory database service provided by Huawei Cloud, supporting mainstream cache engines such as Redis and Memcached. During the use of DCS Redis instances, the system generates background tasks for operations such as instance creation, scaling, configuration changes, backup, and restoration, to track and manage the execution status of these time-consuming operations.

This best practice will introduce how to use Terraform to delete a specified background task of a DCS Redis instance. By calling the Huawei Cloud DCS API for deleting background tasks, you can clean up background task records that are no longer needed, keeping the task list of the instance tidy. This practice operates on a pre-existing DCS instance and background task, and does not create any new infrastructure.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DCS Background Task Deletion (huaweicloud_dcs_background_task_delete)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_background_task_delete)

### Resource/Data Source Dependencies

```
huaweicloud_dcs_background_task_delete
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the scripts of the current best practice in the specified working directory, and ensure that it (or other TF files in the same level directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the article [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Delete the DCS Background Task

Add the following script to the TF file (such as main.tf):

```hcl
# Delete the background task of the DCS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dcs_instance_id" {
  description = "The ID of the DCS instance that owns the background task"
  type        = string
  default     = ""
}

variable "background_task_id" {
  description = "The ID of the background task to delete"
  type        = string
  default     = ""
}

resource "huaweicloud_dcs_background_task_delete" "test" {
  instance_id = var.dcs_instance_id
  task_id     = var.background_task_id
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the input variable dcs_instance_id, specifying the ID of the DCS instance that owns the background task.
- **task_id**: Assigned by referencing the input variable background_task_id, specifying the ID of the background task to delete.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through `tfvars` files, avoiding repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
dcs_instance_id    = "your_dcs_instance_id"
background_task_id = "your_background_task_id"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="dcs_instance_id=my-instance-id"`
2. Environment variables: `export TF_VAR_dcs_instance_id=my-instance-id`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to delete the background task:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource change plan
3. After confirming the resource plan is correct, run `terraform apply` to start deleting the specified background task
4. Run `terraform show` to view the information of the deleted background task

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Background Task Deletion](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-background-task-delete)
