# Deploy Custom Character Mask Algorithm

## Application Scenario

Data Security Center (DSC) is a one-stop data security governance service provided by Huawei Cloud, supporting sensitive data identification, data masking, and data watermarking to help enterprises meet data security compliance requirements. In data masking scenarios, mask algorithms are used to desensitize sensitive fields, ensuring data security during development, testing, and sharing.

This best practice will introduce how to use Terraform to automatically deploy a DSC custom character mask algorithm, which uses the character overwrite combination (`PRESNM` with `MASK_BY_OVERWRITE`) to retain a specified number of characters at the beginning and end of the source data and mask the characters in between with a replacement character.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DSC Mask Algorithm (huaweicloud_dsc_mask_algorithm)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_mask_algorithm)

### Resource/Data Source Dependencies

```
huaweicloud_dsc_mask_algorithm
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a DSC Custom Character Mask Algorithm

Add the following script to the TF file (such as main.tf):

```hcl
# Create a DSC custom character mask algorithm in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "mask_algorithm_name" {
  description = "The name of the mask algorithm"
  type        = string
}

variable "mask_algorithm_prefix_length" {
  description = "The number of characters to retain at the beginning of the data"
  type        = number
  default     = 6
}

variable "mask_algorithm_suffix_length" {
  description = "The number of characters to retain at the end of the data"
  type        = number
  default     = 4
}

variable "mask_algorithm_replacement" {
  description = "The character used to mask the data"
  type        = string
  default     = "*"
}

resource "huaweicloud_dsc_mask_algorithm" "test" {
  algorithm_name = var.mask_algorithm_name
  algorithm      = "PRESNM"
  algorithm_type = "MASK_BY_OVERWRITE"
  category       = "BUILT_SELF"

  parameter = jsonencode({
    type   = "CHAR"
    first  = var.mask_algorithm_prefix_length
    second = var.mask_algorithm_suffix_length
    method = var.mask_algorithm_replacement
  })
}
```

**Parameter Description**:
- **algorithm_name**: The name of the mask algorithm, assigned by referencing the input variable mask_algorithm_name. The name must not be the same as an existing mask algorithm name in the current DSC instance
- **algorithm**: The algorithm type, fixed to `PRESNM`, which retains the characters at the beginning and end of the data
- **algorithm_type**: The algorithm processing method, fixed to `MASK_BY_OVERWRITE`, which masks the data by overwriting
- **category**: The algorithm category, fixed to `BUILT_SELF`, which indicates a custom algorithm
- **parameter**: The algorithm parameters, passed as a JSON string, including the following fields:
  - **type**: The parameter type, fixed to `CHAR`, which indicates the character type
  - **first**: The number of characters to retain at the beginning of the source data, assigned by referencing the input variable mask_algorithm_prefix_length, defaulting to 6
  - **second**: The number of characters to retain at the end of the source data, assigned by referencing the input variable mask_algorithm_suffix_length, defaulting to 4
  - **method**: The replacement character used to mask the data, assigned by referencing the input variable mask_algorithm_replacement, defaulting to `*`

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
mask_algorithm_name = "tfmaskalgorithm"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="mask_algorithm_name=tfmaskalgorithm"`
2. Environment variables: `export TF_VAR_mask_algorithm_name=tfmaskalgorithm`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DSC custom character mask algorithm
4. Run `terraform show` to view the created DSC custom character mask algorithm

## Reference Information

- [Huawei Cloud Data Security Center Product Documentation](https://support.huaweicloud.com/dsc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DSC Custom Character Mask Algorithm](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dsc/custom-character-mask-algorithm)
