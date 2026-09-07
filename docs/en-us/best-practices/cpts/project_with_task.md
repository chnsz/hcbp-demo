# Deploy CPTS Project and Test Task

## Application Scenario

Cloud Performance Test Service (CPTS) is a performance testing service provided by Huawei Cloud. It helps users simulate real business scenarios and perform stress tests on cloud applications to evaluate system performance bottlenecks and stability.

This best practice will introduce how to use Terraform to create a CPTS project and a test task. A CPTS project is used to organize and manage test tasks, while a test task defines parameters such as the concurrency for stress testing. Through automated deployment with Terraform, you can quickly create CPTS projects and test tasks, preparing for subsequent performance tests.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [CPTS Project (huaweicloud_cpts_project)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cpts_project)
- [CPTS Test Task (huaweicloud_cpts_task)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cpts_task)

### Resource/Data Source Dependencies

```
huaweicloud_cpts_project
    └── huaweicloud_cpts_task
```

## Operation Steps

### 1. Script Preparation

Prepare the TF files (such as main.tf) required for writing the best practice scripts in the specified working directory, ensuring that they (or other TF files in the same level directory) contain the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a CPTS Project

Add the following script to the TF file (such as main.tf):

```hcl
# Create a CPTS project in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "cpts_project_name" {
  description = "The name of the CPTS project"
  type        = string
}

variable "cpts_project_description" {
  description = "The description of the CPTS project"
  type        = string
  default     = ""
}

resource "huaweicloud_cpts_project" "test" {
  name        = var.cpts_project_name
  description = var.cpts_project_description
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable cpts_project_name, used to specify the name of the CPTS project.
- **description**: Assigned by referencing the input variable cpts_project_description, used to describe the CPTS project.

### 3. Create a CPTS Test Task

Add the following script to the TF file (such as main.tf):

```hcl
# Create a CPTS test task in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "cpts_task_name" {
  description = "The name of the CPTS test task"
  type        = string
}

variable "cpts_task_benchmark_concurrency" {
  description = "The benchmark concurrency of the CPTS test task"
  type        = number
  default     = 200
}

resource "huaweicloud_cpts_task" "test" {
  name                  = var.cpts_task_name
  project_id            = huaweicloud_cpts_project.test.id
  benchmark_concurrency = var.cpts_task_benchmark_concurrency
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable cpts_task_name, used to specify the name of the CPTS test task.
- **project_id**: Assigned by referencing huaweicloud_cpts_project.test.id, used to specify the ID of the CPTS project to which the test task belongs.
- **benchmark_concurrency**: Assigned by referencing the input variable cpts_task_benchmark_concurrency, used to specify the benchmark concurrency of the test task.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this best practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
region_name                     = "cn-north-4"
access_key                      = "your-access-key"
secret_key                      = "your-secret-key"
cpts_project_name               = "tf_test_cpts_project"
cpts_project_description        = "Created by Terraform for CPTS best practice"
cpts_task_name                  = "tf_test_cpts_task"
cpts_task_benchmark_concurrency = 200
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="cpts_project_name=my-project"`
2. Environment variables: `export TF_VAR_cpts_project_name=my-project`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the CPTS project and test task
4. Run `terraform show` to view the created CPTS project and test task

## Reference Information

- [Huawei Cloud CPTS Product Documentation](https://support.huaweicloud.com/cpts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CPTS Project and Test Task](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cpts/project-with-task)
