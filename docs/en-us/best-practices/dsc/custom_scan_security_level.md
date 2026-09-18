# Deploy Custom Scan Security Level

## Application Scenario

Data Security Center (DSC) is a one-stop data security governance service provided by Huawei Cloud, supporting sensitive data scanning, classification, and grading for various data sources such as cloud databases, big data services, and object storage. During sensitive data identification, security levels are used to grade the identified sensitive data, helping users define differentiated protection and governance policies based on data sensitivity.

This best practice will introduce how to use Terraform to automatically deploy a DSC custom scan security level, including the configuration of the security level name, the color number displayed on the console, and the security level description.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DSC Scan Security Level (huaweicloud_dsc_scan_security_level)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_scan_security_level)

### Resource/Data Source Dependencies

```
huaweicloud_dsc_scan_security_level
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration details.

### 2. Create a DSC Scan Security Level

Add the following script to the TF file (such as main.tf) to create a DSC scan security level:

```hcl
# Create a DSC scan security level in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_level_name" {
  description = "The name of the security level"
  type        = string
}

variable "security_level_color_number" {
  description = "The color number of the security level displayed on the console"
  type        = number
  default     = 6
}

variable "security_level_description" {
  description = "The description of the security level"
  type        = string
  default     = ""
}

resource "huaweicloud_dsc_scan_security_level" "test" {
  security_level_name = var.security_level_name
  color_number        = var.security_level_color_number
  security_level_desc = var.security_level_description
}
```

**Parameter Description**:

- **security_level_name**: The name of the security level, assigned by referencing the input variable security_level_name. The name must not be the same as an existing security level name in the current DSC instance
- **color_number**: The color number of the security level displayed on the console, assigned by referencing the input variable security_level_color_number. The default value is 6
- **security_level_desc**: The description of the security level, assigned by referencing the input variable security_level_description. The default value is an empty string

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through the `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# The name of the security level
security_level_name = "tfleveltest"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="security_level_name=tfleveltest"`
2. Environment variables: `export TF_VAR_security_level_name=tfleveltest`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DSC scan security level
4. Run `terraform show` to view the created DSC scan security level

## Reference Information

- [Huawei Cloud Data Security Center Product Documentation](https://support.huaweicloud.com/dsc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DSC Custom Scan Security Level](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dsc/custom-scan-security-level)
