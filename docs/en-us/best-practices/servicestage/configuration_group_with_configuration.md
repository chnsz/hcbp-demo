# Deploy Configuration Group with Configuration

## Application Scenario

ServiceStage is a one-stop application management service provided by Huawei Cloud, supporting the full lifecycle management of applications, including development, build, deployment, governance, and operations. In microservice and cloud-native scenarios, configuration management is a key part of application deployment and runtime. Configuration groups and configuration files allow you to centrally manage environment variables, startup parameters, and business configurations, separating configuration from code.

This best practice will introduce how to use Terraform to automatically create a ServiceStage configuration group and its configuration file, helping you efficiently manage application configurations in an Infrastructure as Code (IaC) manner and laying a solid foundation for subsequent application deployment and configuration updates.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [ServiceStage Configuration Group (huaweicloud_servicestagev3_configuration_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_configuration_group)
- [ServiceStage Configuration (huaweicloud_servicestagev3_configuration)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_configuration)

### Resource/Data Source Dependencies

```
huaweicloud_servicestagev3_configuration_group
    └── huaweicloud_servicestagev3_configuration
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration details.

### 2. Create a ServiceStage Configuration Group

Add the following script to the TF file (such as main.tf) to create a configuration group:

```hcl
# Create a ServiceStage configuration group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **name**: The name of the configuration group, assigned by referencing the input variable configuration_group_name. The name must contain 2 to 64 characters, only letters, digits, hyphens (-), and underscores (_) are allowed, and it must start with a letter and end with a letter or a digit
- **description**: The description of the configuration group, assigned by referencing the input variable configuration_group_description. This parameter is optional

### 3. Create a ServiceStage Configuration

Add the following script to the TF file (such as main.tf) to create a configuration file that belongs to the configuration group above:

```hcl
# Create a ServiceStage configuration in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **config_group_id**: The ID of the configuration group to which the configuration file belongs, assigned by referencing the resource huaweicloud_servicestagev3_configuration_group.test.id
- **name**: The name of the configuration file, assigned by referencing the input variable configuration_name. The naming rules are the same as those of the configuration group name
- **type**: The type of the configuration file, currently only `yaml` and `properties` are supported. This practice uses `properties`, and the content must match the declared type
- **content**: The content of the configuration file, assigned by referencing the input variable configuration_content. ServiceStage system variables are supported and referenced by `$${VARIABLE_NAME}`
- **description**: The description of the configuration file, assigned by referencing the input variable configuration_description. This parameter is optional
- **lifecycle.ignore_changes**: Since the query detail API does not return the type attribute, changes to the type argument are ignored here to avoid a permanent in-place update diff. If you need to change the type, remove it from ignore_changes first

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

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="configuration_group_name=my-config-group"`
2. Environment variables: `export TF_VAR_configuration_group_name=my-config-group`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the ServiceStage configuration group and configuration file
4. Run `terraform show` to view the created ServiceStage configuration group and configuration file

## Reference Information

- [Huawei Cloud ServiceStage Product Documentation](https://support.huaweicloud.com/servicestage/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ServiceStage Configuration Group with Configuration](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/servicestage/configuration-group-with-configuration)
