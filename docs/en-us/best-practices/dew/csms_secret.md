# Deploy CSMS Secret

## Application Scenario

Data Encryption Workshop (DEW) CSMS (Cloud Secret Management Service) secret is a secret management function provided by the DEW service, used to securely store and manage sensitive information such as passwords, API keys, certificates, etc. By creating CSMS secrets, you can store sensitive information in the cloud, achieve unified management and secure access to secrets, avoid hardcoding sensitive information in code or configuration files, and improve application security. Automating CSMS secret creation through Terraform can ensure standardized and consistent secret configuration, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create CSMS secrets.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [CSMS Secret Resource (huaweicloud_csms_secret)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/csms_secret)

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create CSMS Secret Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CSMS secret resource:

```hcl
variable "secret_name" {
  description = "The name of the secret"
  type        = string
}

variable "secret_text" {
  description = "The plaintext of a text secret"
  type        = string
  sensitive   = true
}

variable "secret_type" {
  description = "The type of the secret"
  type        = string
  default     = "COMMON"
}

variable "kms_key_id" {
  description = "The ID of the KMS key used to encrypt the secret"
  type        = string
  default     = ""
}

variable "secret_description" {
  description = "The description of the secret"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the secret belongs"
  type        = string
  default     = null
}

variable "secret_tags" {
  description = "The key/value pairs to associate with the secret"
  type        = map(string)
  default     = {}
}

# Create CSMS secret resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_csms_secret" "test" {
  name                  = var.secret_name
  secret_text           = var.secret_text
  secret_type           = var.secret_type
  kms_key_id            = var.kms_key_id != "" ? var.kms_key_id : null
  description           = var.secret_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.secret_tags
}
```

**Parameter Description**:
- **name**: The secret name, assigned by referencing the input variable secret_name
- **secret_text**: The plaintext of the secret, assigned by referencing the input variable secret_text
- **secret_type**: The secret type, assigned by referencing the input variable secret_type, default value is "COMMON" (common secret)
- **kms_key_id**: The KMS key ID used to encrypt the secret, assigned by referencing the input variable kms_key_id, optional parameter, default value is null (use default KMS key)
- **description**: The secret description, assigned by referencing the input variable secret_description, optional parameter, default value is empty string
- **enterprise_project_id**: The enterprise project ID to which the secret belongs, assigned by referencing the input variable enterprise_project_id, optional parameter, default value is null
- **tags**: The secret tags, assigned by referencing the input variable secret_tags, optional parameter, default value is empty map

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# CSMS Secret Configuration
secret_name           = "tf_test_secret"
secret_text           = "your_secret_text"
secret_description    = "This is a CSMS secret created by Terraform"
enterprise_project_id = "0"

# Secret Tags Configuration
secret_tags = {
  owner = "terraform"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially `secret_text` needs to be replaced with the actual secret content
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="secret_name=my_secret" -var="secret_text=my_secret_text"`
2. Environment variables: `export TF_VAR_secret_name=my_secret` and `export TF_VAR_secret_text=my_secret_text`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values. Since secret_text contains sensitive information, it is recommended to use environment variables or encrypted variable files for setting.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a CSMS secret:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CSMS secret
4. Run `terraform show` to view the details of the created CSMS secret

> Note: After the CSMS secret is created, sensitive information will be encrypted and stored in the cloud, and can be securely accessed through APIs or the console. Secrets support encryption using KMS keys, providing higher security. Secret tags can help you categorize and manage secrets. Please ensure that secret information is properly kept and do not commit sensitive information to version control systems.

## Reference Information

- [Huawei Cloud DEW Product Documentation](https://support.huaweicloud.com/dew/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CSMS Secret](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dew/csms-secret)
