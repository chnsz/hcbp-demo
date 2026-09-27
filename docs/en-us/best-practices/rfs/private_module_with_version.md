# Deploy Private Module with Version

## Application Scenario

Resource Formation Service (RFS) is an Infrastructure as Code (IaC) service provided by Huawei Cloud, supporting the orchestration and automated deployment of cloud resources through templates. A private module packages a set of reusable templates into a standardized module, and module versions enable versioned management of these templates, making it easy to share and reuse infrastructure code across teams and projects.

This best practice will introduce how to use Terraform to create an RFS private module and create a corresponding module version based on a module package stored in OBS, achieving automated deployment and management of the private module and its version.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [RFS Private Module (huaweicloud_rfs_private_module)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_private_module)
- [RFS Private Module Version (huaweicloud_rfs_private_module_version)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_private_module_version)

### Resource/Data Source Dependencies

```
huaweicloud_rfs_private_module
    └── huaweicloud_rfs_private_module_version
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create an RFS Private Module

Add the following script in the TF file (such as main.tf) to create an RFS private module:

```hcl
# Create an RFS private module resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "module_name" {
  description = "The name of the RFS private module"
  type        = string
}

variable "module_description" {
  description = "The description of the RFS private module"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_private_module" "test" {
  module_name        = var.module_name
  module_description = var.module_description
}
```

**Parameter Description**:
- **module_name**: The name of the private module, assigned by referencing the input variable module_name. It must be unique within the current account and region, only letters, digits, underscores (_) and hyphens (-) are allowed, and it must start with a letter
- **module_description**: The description of the private module, assigned by referencing the input variable module_description. This parameter is optional, and once set it cannot be updated to an empty value

### 3. Create an RFS Private Module Version

Add the following script in the TF file (such as main.tf) to create an RFS private module version:

```hcl
# Create an RFS private module version resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "module_version" {
  description = "The version number of the RFS private module"
  type        = string
}

variable "module_uri" {
  description = "The OBS address of the private module package"
  type        = string
}

variable "version_description" {
  description = "The description of the private module version"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_private_module_version" "test" {
  module_name         = huaweicloud_rfs_private_module.test.module_name
  module_version      = var.module_version
  module_uri          = var.module_uri
  version_description = var.version_description
}
```

**Parameter Description**:
- **module_name**: The name of the private module, assigned by referencing the module_name attribute of the private module created in the previous step
- **module_version**: The version number of the private module, assigned by referencing the input variable module_version. This parameter is non-updatable, and changing it will recreate the module version
- **module_uri**: The OBS address of the private module package, assigned by referencing the input variable module_uri. This parameter is non-updatable, and changing it will recreate the module version
- **version_description**: The description of the private module version, assigned by referencing the input variable version_description. This parameter is optional and non-updatable

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Private module configuration
module_name        = "tf-test-rfs-module"
module_description = "An example RFS private module"

# Private module version configuration
module_version      = "1.0.0"
module_uri          = "https://your-bucket.obs.cn-north-4.myhuaweicloud.com/tf-test-rfs-module.zip"
version_description = "The first version of the RFS private module"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="module_name=my-module"`
2. Environment variables: `export TF_VAR_module_name=my-module`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the RFS private module and module version
4. Run `terraform show` to view the created RFS private module and module version

## Reference Information

- [Huawei Cloud Resource Formation Service (RFS) Product Documentation](https://support.huaweicloud.com/rfs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For RFS Private Module with Version](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rfs/private-module-with-version)
