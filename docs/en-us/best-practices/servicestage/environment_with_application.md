# Deploy Environment with Application and Its Application Configuration

## Application Scenario

ServiceStage is a one-stop application hosting platform provided by Huawei Cloud, delivering full lifecycle management capabilities for enterprise and cloud-native applications, covering application development, build, deployment, and operations. When migrating applications to the cloud or building new cloud-native applications, you usually need to prepare the network environment that hosts the applications, then create the application runtime environment and the application, and configure runtime parameters such as environment variables for the application.

This best practice will introduce how to use Terraform to automatically create a VPC, a ServiceStage environment bound to the VPC, an application, and the application-level environment variable configuration, helping you quickly build the foundation for application hosting in an Infrastructure as Code (IaC) manner and laying a solid foundation for subsequent component deployment and operations work.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [ServiceStage Environment (huaweicloud_servicestagev3_environment)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_environment)
- [ServiceStage Application (huaweicloud_servicestagev3_application)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_application)
- [ServiceStage Application Configuration (huaweicloud_servicestagev3_application_configuration)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_application_configuration)

### Resource/Data Source Dependencies

```
huaweicloud_vpc
    └── huaweicloud_servicestagev3_environment
            └── huaweicloud_servicestagev3_application_configuration
huaweicloud_servicestagev3_application
    └── huaweicloud_servicestagev3_application_configuration
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Virtual Private Cloud

Add the following script in the TF file (such as main.tf):

```hcl
# Create a virtual private cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = "192.168.0.0/16"
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, fixed to `192.168.0.0/16` here

### 3. Create a ServiceStage Environment

Add the following script in the TF file (such as main.tf):

```hcl
# Create a ServiceStage environment resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "environment_name" {
  description = "The name of the environment"
  type        = string
}

variable "environment_description" {
  description = "The description of the environment"
  type        = string
  default     = ""
}

resource "huaweicloud_servicestagev3_environment" "test" {
  name        = var.environment_name
  description = var.environment_description
  vpc_id      = huaweicloud_vpc.test.id
}
```

**Parameter Description**:
- **name**: The environment name, assigned by referencing the input variable environment_name; it must contain 2 to 64 characters, only letters, digits, hyphens (-) and underscores (_) are allowed, and it must start with a letter and end with a letter or a digit
- **description**: The environment description, assigned by referencing the input variable environment_description; it must not exceed 128 characters
- **vpc_id**: The ID of the VPC bound to the environment, assigned by referencing the ID of the VPC created in the previous step; this parameter cannot be updated, and changing it will recreate the environment

### 4. Create a ServiceStage Application

Add the following script in the TF file (such as main.tf):

```hcl
# Create a ServiceStage application resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "application_name" {
  description = "The name of the application"
  type        = string
}

variable "application_description" {
  description = "The description of the application"
  type        = string
  default     = ""
}

resource "huaweicloud_servicestagev3_application" "test" {
  name        = var.application_name
  description = var.application_description
}
```

**Parameter Description**:
- **name**: The application name, assigned by referencing the input variable application_name; it must contain 2 to 64 characters, only letters, digits, hyphens (-) and underscores (_) are allowed, and it must start with a letter and end with a letter or a digit
- **description**: The application description, assigned by referencing the input variable application_description; it must not exceed 128 characters

### 5. Create an Application Configuration

Add the following script in the TF file (such as main.tf):

```hcl
# Create a ServiceStage application configuration resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "application_configuration_env_name" {
  description = "The name of the environment variable"
  type        = string
}

variable "application_configuration_env_value" {
  description = "The value of the environment variable"
  type        = string
}

resource "huaweicloud_servicestagev3_application_configuration" "test" {
  environment_id = huaweicloud_servicestagev3_environment.test.id
  application_id = huaweicloud_servicestagev3_application.test.id

  configuration {
    env {
      name  = var.application_configuration_env_name
      value = var.application_configuration_env_value
    }
  }
}
```

**Parameter Description**:
- **environment_id**: The ID of the environment to which the application configuration belongs, assigned by referencing the ID of the ServiceStage environment created in the previous step; this parameter cannot be updated, and changing it will recreate the resource
- **application_id**: The ID of the application to which the application configuration belongs, assigned by referencing the ID of the ServiceStage application created in the previous step; this parameter cannot be updated, and changing it will recreate the resource
- **configuration.env.name**: The environment variable name, assigned by referencing the input variable application_configuration_env_name; it must contain 1 to 64 characters and must start with a letter, a hyphen (-) or an underscore (_)
- **configuration.env.value**: The environment variable value, assigned by referencing the input variable application_configuration_env_value

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through the `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
vpc_name                            = "tf_test_ss_vpc"
environment_name                    = "tf_test_ss_environment"
environment_description             = "Created by Terraform for ServiceStage best practice example"
application_name                    = "tf_test_ss_application"
application_description             = "Created by Terraform for ServiceStage best practice example"
application_configuration_env_name  = "TF_EXAMPLE_APP_ENV"
application_configuration_env_value = "tf_example_value"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the ServiceStage environment, application, and application configuration
4. Run `terraform show` to view the created ServiceStage environment, application, and application configuration

## Reference Information

- [Huawei Cloud ServiceStage Product Documentation](https://support.huaweicloud.com/servicestage/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ServiceStage Environment with Application and Its Application Configuration](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/servicestage/environment-with-application)
