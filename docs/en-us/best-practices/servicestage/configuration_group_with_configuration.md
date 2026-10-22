# Deploy Configuration Group with Configuration File

## Application Scenario

ServiceStage is a one-stop application hosting platform provided by Huawei Cloud, supporting the full lifecycle management of applications from development and deployment to operations. In the deployment of microservice and containerized applications, configuration management is key to ensuring application portability and environment isolation. By grouping configuration files with configuration groups, you can centrally maintain the configurations of the same application or environment and reuse them across environments.

This best practice will introduce how to use Terraform to automatically deploy a ServiceStage configuration group and its configuration file, including creating the configuration group and creating the configuration file with its content, helping you efficiently manage application configurations in an Infrastructure as Code (IaC) manner.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Configuration Group (huaweicloud_servicestagev3_configuration_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_configuration_group)
- [Configuration File (huaweicloud_servicestagev3_configuration)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_configuration)

### Resource/Data Source Dependencies

```
huaweicloud_servicestagev3_configuration_group
    └── huaweicloud_servicestagev3_configuration
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a Configuration Group

Add the following script in the TF file (such as main.tf) to create a ServiceStage configuration group:

```hcl
# Create a ServiceStage configuration group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "configuration_group_name" {
  description = "The name of the configuration group"
  type        = string
}

variable "configuration_group_description" {
  description = "The description of the configuration group"
  type        = string
  default     = ""
}

resource "huaweicloud_servicestagev3_configuration_group" "test" {
  name        = var.configuration_group_name
  description = var.configuration_group_description
}
```

**Parameter Description**:

- **name**: The name of the configuration group, assigned by referencing the input variable `configuration_group_name`. The name must contain 2 to 64 characters, only letters, digits, hyphens (-) and underscores (_) are allowed, and it must start with a letter and end with a letter or a digit
- **description**: The description of the configuration group, assigned by referencing the input variable `configuration_group_description`. This parameter is optional

### 3. Create a Configuration File

Add the following script in the TF file (such as main.tf) to create a configuration file under the configuration group:

```hcl
# Create a ServiceStage configuration file resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "configuration_name" {
  description = "The name of the configuration file"
  type        = string
}

variable "configuration_content" {
  description = "The content of the configuration file"
  type        = string
}

variable "configuration_description" {
  description = "The description of the configuration file"
  type        = string
  default     = ""
}

resource "huaweicloud_servicestagev3_configuration" "test" {
  config_group_id = huaweicloud_servicestagev3_configuration_group.test.id
  name            = var.configuration_name
  type            = "properties"
  content         = var.configuration_content
  description     = var.configuration_description

  lifecycle {
    ignore_changes = [
      type
    ]
  }
}
```

**Parameter Description**:

- **config_group_id**: The ID of the configuration group to which the configuration file belongs, assigned by referencing `huaweicloud_servicestagev3_configuration_group.test.id`
- **name**: The name of the configuration file, assigned by referencing the input variable `configuration_name`. The naming rules are the same as those of the configuration group
- **type**: The type of the configuration file, set to `properties`. Only `yaml` and `properties` are supported, and the `content` must match the declared type
- **content**: The content of the configuration file, assigned by referencing the input variable `configuration_content`. ServiceStage system variables can be referenced by `$${VARIABLE_NAME}`
- **description**: The description of the configuration file, assigned by referencing the input variable `configuration_description`. This parameter is optional
- **lifecycle.ignore_changes**: The query detail API does not return the `type` attribute, so `type` is ignored here to avoid a permanent in-place update diff. If you need to change `type`, remove it from `ignore_changes` first

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
configuration_group_name        = "tf_test_ss_config_group"
configuration_group_description = "Created by Terraform for ServiceStage best practice example"
configuration_name              = "tf_test_ss_configuration"
configuration_content           = "spring.application.name = tf-example-service"
configuration_description       = "Created by Terraform for ServiceStage best practice example"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="configuration_group_name=my-config-group"`
2. Environment variables: `export TF_VAR_configuration_group_name=my-config-group`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the ServiceStage configuration group and configuration file
4. Run `terraform show` to view the created ServiceStage configuration group and configuration file

## Reference Information

- [Huawei Cloud ServiceStage Product Documentation](https://support.huaweicloud.com/servicestage/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ServiceStage Configuration Group with Configuration File](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/servicestage/configuration-group-with-configuration)
