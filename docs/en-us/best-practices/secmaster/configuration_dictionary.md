# Deploy Configuration Dictionary

## Application Scenario

SecMaster is a next-generation cloud native security operations center provided by Huawei Cloud, offering capabilities such as cloud asset management, security posture management, security information and event management, and security orchestration and automatic response. A configuration dictionary maintains the set of selectable enumeration values for business objects such as alerts and incidents, for example alert comment status and handling conclusions, keeping field values consistent and standardized across security operations processes.

This best practice will introduce how to use Terraform to automatically deploy a SecMaster configuration dictionary, including basic dictionary information, language environment, version number, parent dictionary, and extension fields.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [SecMaster Configuration Dictionary (huaweicloud_secmaster_configuration_dictionary)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_configuration_dictionary)

### Resource/Data Source Dependencies

```
huaweicloud_secmaster_configuration_dictionary
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a SecMaster Configuration Dictionary

Add the following script in the TF file (such as main.tf) to create a SecMaster configuration dictionary:

```hcl
# Create a SecMaster configuration dictionary resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dict_id" {
  description = "The dictionary ID"
  type        = string
}

variable "dict_key" {
  description = "The dictionary key"
  type        = string
}

variable "dict_code" {
  description = "The dictionary code"
  type        = string
}

variable "dict_val" {
  description = "The dictionary value"
  type        = string
}

variable "language" {
  description = "The language environment. Valid values: zh, en"
  type        = string
  default     = "zh"
}

variable "dict_version" {
  description = "The version number"
  type        = string
  default     = "1.0.0"
}

variable "dict_pkey" {
  description = "The parent key of the dictionary"
  type        = string
  default     = ""
}

variable "dict_pcode" {
  description = "The parent code of the dictionary"
  type        = string
  default     = ""
}

variable "scope" {
  description = "The domain to which the dictionary belongs"
  type        = string
  default     = "ALERT"
}

variable "description" {
  description = "The description of the dictionary"
  type        = string
  default     = ""
}

variable "extend_field" {
  description = "The extension field of the dictionary"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_secmaster_configuration_dictionary" "test" {
  dict_id      = var.dict_id
  dict_key     = var.dict_key
  dict_code    = var.dict_code
  dict_val     = var.dict_val
  language     = var.language
  version      = var.dict_version
  dict_pkey    = var.dict_pkey
  dict_pcode   = var.dict_pcode
  scope        = var.scope
  description  = var.description
  extend_field = var.extend_field
  is_built_in  = false

  lifecycle {
    ignore_changes = [
      is_built_in,
    ]
  }
}
```

**Parameter Description**:
- **dict_id**: The dictionary ID, assigned by referencing the input variable dict_id, and cannot be modified after creation
- **dict_key**: The dictionary key, assigned by referencing the input variable dict_key, and cannot be modified after creation
- **dict_code**: The dictionary code, assigned by referencing the input variable dict_code
- **dict_val**: The dictionary value, assigned by referencing the input variable dict_val
- **language**: The language environment, assigned by referencing the input variable language, valid values are zh and en, and cannot be modified after creation
- **version**: The version number, assigned by referencing the input variable dict_version, and cannot be modified after creation
- **dict_pkey**: The parent key of the dictionary, assigned by referencing the input variable dict_pkey
- **dict_pcode**: The parent code of the dictionary, assigned by referencing the input variable dict_pcode
- **scope**: The domain to which the dictionary belongs, assigned by referencing the input variable scope, and cannot be modified after creation
- **description**: The description of the dictionary, assigned by referencing the input variable description
- **extend_field**: The extension field of the dictionary, assigned by referencing the input variable extend_field
- **is_built_in**: Whether the dictionary is built-in, set to false here to create a user-defined dictionary; this attribute may drift after import, so its changes are ignored via lifecycle ignore_changes

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Configuration dictionary variables
dict_id   = "3027"
dict_key  = "alert_comments"
dict_code = "Open"
dict_val  = "Open"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="dict_id=3027"`
2. Environment variables: `export TF_VAR_dict_id=3027`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the SecMaster configuration dictionary
4. Run `terraform show` to view the created SecMaster configuration dictionary

## Reference Information

- [Huawei Cloud SecMaster Product Documentation](https://support.huaweicloud.com/intl/en-us/secmaster/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For SecMaster Configuration Dictionary](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/configuration-dictionary)
