# Deploy Resource Stack with Execution Plan

## Application Scenario

Resource Formation Service (RFS) is an Infrastructure as Code (IaC) service provided by Huawei Cloud, allowing users to define and orchestrate cloud resources in a template-based manner. In real delivery scenarios, users often want to preview the resource changes that a template will bring before actually creating resources, so as to reduce the risk of misoperations. The RFS execution plan is designed for this purpose: it generates a preview of resource changes based on an inline Terraform template body without actually creating any cloud resources.

This best practice will introduce how to use Terraform to create an empty RFS resource stack and then create an execution plan for it based on an inline Terraform template body, so that resource changes can be previewed before deployment. Since the resource stack itself carries no template, no cloud resources will be deployed and no agency authorization is required; the execution plan is only used to preview the resource changes described by the template body.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Resource Stack (huaweicloud_rfs_stack)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_stack)
- [Execution Plan (huaweicloud_rfs_execution_plan_v2)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_execution_plan_v2)

### Resource/Data Source Dependencies

```
huaweicloud_rfs_stack
    └── huaweicloud_rfs_execution_plan_v2
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Resource Stack

Add the following script in the TF file (such as main.tf):

```hcl
# Create a resource stack resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "stack_name" {
  description = "The name of the RFS resource stack"
  type        = string
}

variable "stack_description" {
  description = "The description of the RFS resource stack"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_stack" "test" {
  name        = var.stack_name
  description = var.stack_description
}
```

**Parameter Description**:
- **name**: The name of the resource stack, assigned by referencing the input variable stack_name
- **description**: The description of the resource stack, assigned by referencing the input variable stack_description, which defaults to an empty string

### 3. Create an Execution Plan

Add the following script in the TF file (such as main.tf):

```hcl
# Create an execution plan resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "execution_plan_name" {
  description = "The name of the RFS execution plan"
  type        = string
}

variable "execution_plan_description" {
  description = "The description of the RFS execution plan"
  type        = string
  default     = ""
}

variable "execution_plan_template_body" {
  description = "The Terraform template body used by the RFS execution plan"
  type        = string
}

resource "huaweicloud_rfs_execution_plan_v2" "test" {
  stack_name          = huaweicloud_rfs_stack.test.name
  stack_id            = huaweicloud_rfs_stack.test.id
  execution_plan_name = var.execution_plan_name
  description         = var.execution_plan_description
  template_body       = var.execution_plan_template_body
}
```

**Parameter Description**:
- **stack_name**: The name of the resource stack to which the execution plan belongs, assigned by referencing the name attribute of the resource stack created in the previous step
- **stack_id**: The ID of the resource stack to which the execution plan belongs, assigned by referencing the id attribute of the resource stack created in the previous step
- **execution_plan_name**: The name of the execution plan, assigned by referencing the input variable execution_plan_name
- **description**: The description of the execution plan, assigned by referencing the input variable execution_plan_description, which defaults to an empty string
- **template_body**: The Terraform template body used by the execution plan, assigned by referencing the input variable execution_plan_template_body. The template body must declare a `required_providers` block with the provider source set to `huawei.com/provider/huaweicloud`; otherwise the execution plan engine cannot resolve the provider packages and the execution plan creation will fail

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
stack_name          = "tf-test-stack"
execution_plan_name = "tf-test-plan"

execution_plan_template_body = <<-EOT
terraform {
  required_providers {
    huaweicloud = {
      source  = "huawei.com/provider/huaweicloud"
      version = ">= 1.41.0"
    }
  }
}

resource "huaweicloud_vpc" "test" {
  name = "tf-test-vpc"
  cidr = "192.168.0.0/16"
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = "tf-test-subnet"
  cidr       = "192.168.1.0/24"
  gateway_ip = "192.168.1.1"
}
EOT
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="stack_name=my-stack"`
2. Environment variables: `export TF_VAR_stack_name=my-stack`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the resource stack and execution plan
4. Run `terraform show` to view the created resource stack and execution plan

## Reference Information

- [Huawei Cloud Resource Formation Service (RFS) Product Documentation](https://support.huaweicloud.com/rfs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For RFS Resource Stack with Execution Plan](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rfs/stack-with-execution-plan)
