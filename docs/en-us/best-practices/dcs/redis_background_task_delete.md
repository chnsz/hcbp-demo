# Deploy Redis Background Task Deletion

## Application Scenario

Distributed Cache Service (DCS) is a high-performance, highly available in-memory database service provided by Huawei Cloud, supporting mainstream cache engines such as Redis and Memcached. During routine O&M of DCS instances, the system automatically generates background tasks during operations such as instance creation, scaling, configuration changes, backup, and restoration, and these tasks are recorded in the instance's task list.

This best practice will introduce how to use Terraform to delete a specified background task of a DCS Redis instance. This practice operates on a pre-existing DCS instance and background task, and does not create any new infrastructure. It is suitable for scenarios where you need to clean up completed or abnormal background tasks and keep the task list tidy.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DCS Background Task Deletion Resource (huaweicloud_dcs_background_task_delete)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_background_task_delete)

### Resource/Data Source Dependencies

```
huaweicloud_dcs_background_task_delete
```

## Operation Steps

### 1. Script Preparation

Prepare the TF files (such as main.tf) required for writing the current best practice script in the specified working directory, and ensure that they (or other TF files in the same level directory) contain the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

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

> Note: This resource is a delete-action resource, and its behavior differs fundamentally from standard Terraform resources. The create operation sends a DELETE request to the DCS API to remove the specified background task, the read and update operations do not perform any API calls, and the delete operation only removes the resource from the Terraform state without restoring the deleted background task.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through `tfvars` files, avoiding repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
dcs_instance_id    = "your_dcs_instance_id"
background_task_id = "your_background_task_id"
```

**Usage**:

1. Save the above content as the `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="dcs_instance_id=my-instance-id"`
2. Environment variables: `export TF_VAR_dcs_instance_id=my-instance-id`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to delete the background task:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start deleting the background task
4. Run `terraform show` to view the deleted background task

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Background Task Deletion](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-background-task-delete)
