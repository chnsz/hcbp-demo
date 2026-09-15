# Deploy Parameter Template

## Application Scenario

Document Database Service (DDS) allows you to manage database parameters of instances in a unified way through parameter templates. You can save a set of validated parameter configurations as a template and quickly apply it when creating or changing instances, ensuring consistency and maintainability of parameter configurations across different instances.

This best practice will introduce how to use Terraform to automatically deploy a DDS parameter template, including the creation of the parameter template and its name, node type, database version, parameter mapping, and description.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DDS Parameter Template (huaweicloud_dds_parameter_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_parameter_template)

### Resource/Data Source Dependencies

```
huaweicloud_dds_parameter_template
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a DDS Parameter Template

Add the following script in the TF file (such as main.tf) to create a DDS parameter template:

```hcl
# Create a DDS parameter template resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "template_name" {
  description = "The name of the parameter template"
  type        = string
}

variable "template_mapping" {
  description = "The mapping between parameter names and parameter values"
  type        = map(string)
}

variable "template_node_type" {
  description = "The node type of the parameter template"
  type        = string
  default     = "mongos"
}

variable "database_version" {
  description = "The database version"
  type        = string
  default     = "4.0"
}

variable "template_description" {
  description = "The description of the parameter template"
  type        = string
  default     = ""
}

resource "huaweicloud_dds_parameter_template" "test" {
  name             = var.template_name
  parameter_values = var.template_mapping
  node_type        = var.template_node_type
  node_version     = var.database_version
  description      = var.template_description
}
```

**Parameter Description**:
- **name**: The name of the parameter template, assigned by referencing the input variable template_name
- **parameter_values**: The mapping between parameter names and parameter values, assigned by referencing the input variable template_mapping
- **node_type**: The node type of the parameter template, assigned by referencing the input variable template_node_type. The value can be `mongos`, `shard`, `config`, `replica`, `readonly`, `shard_readonly`, or `single`
- **node_version**: The database version, assigned by referencing the input variable database_version. The value can be `5.0`, `4.4`, `4.2`, `4.0`, or `3.4`
- **description**: The description of the parameter template, assigned by referencing the input variable template_description

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
template_name    = "tf_test_template"
template_mapping = {
  connPoolMaxConnsPerHost        = 500
  connPoolMaxShardedConnsPerHost = 500
}
template_description = "This is a parameter template created by terraform"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="template_name=my-template"`
2. Environment variables: `export TF_VAR_template_name=my-template`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DDS parameter template
4. Run `terraform show` to view the created DDS parameter template

## Reference Information

- [Huawei Cloud Document Database Service Product Documentation](https://support.huaweicloud.com/dds/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DDS Parameter Template](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/parameter-template)
