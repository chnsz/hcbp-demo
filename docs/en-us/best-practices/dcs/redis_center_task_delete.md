# Deploy Redis Center Task Deletion

## Application Scenario

Distributed Cache Service (DCS) is a high-performance, highly available in-memory database service provided by Huawei Cloud, supporting mainstream cache engines such as Redis and Memcached. During the use of DCS Redis instances, some background tasks may be generated, such as instance specification changes, backup and restoration, etc. These tasks will remain in the task list after completion.

This best practice will introduce how to use Terraform to delete a specified center task of a DCS Redis instance, suitable for cleaning up completed or abnormal center tasks, helping you keep the task list tidy and achieve automated management of DCS center tasks.

## Related Resources

This best practice involves the following main resources:

### Resources

- [DCS Center Task Deletion (huaweicloud_dcs_center_task_delete)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_center_task_delete)

### Resource/Data Source Dependencies

```
huaweicloud_dcs_center_task_delete
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified working directory, ensuring that it (or other TF files in the same level directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Delete the DCS Center Task

Add the following script to the TF file (such as main.tf):

```hcl
# Delete the DCS center task in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "task_id" {
  description = "The ID of the DCS center task to be deleted"
  type        = string
}

resource "huaweicloud_dcs_center_task_delete" "test" {
  task_id = var.task_id
}
```

**Parameter Description**:
- **task_id**: Assigned by referencing the input variable task_id, specifying the ID of the DCS center task to be deleted.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to the configuration. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory. The example content is as follows:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
task_id = "your_dcs_center_task_id"
```

**Usage**:

1. Save the above content as the `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="task_id=your_dcs_center_task_id"`
2. Environment variables: `export TF_VAR_task_id=your_dcs_center_task_id`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to delete the DCS center task
4. Run `terraform show` to view the deleted DCS center task

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Center Task Deletion](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-center-task-delete)
