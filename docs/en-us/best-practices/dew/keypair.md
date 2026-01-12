# Deploy Keypair

## Application Scenario

Data Encryption Workshop (DEW) KPS (Key Pair Service) keypair is a key pair management function provided by the DEW service, used to create and manage SSH key pairs, providing secure login authentication for cloud resources such as ECS instances. By creating KPS keypairs, you can generate SSH key pairs, inject public keys into ECS instances, achieve passwordless login, and improve security and convenience. Automating KPS keypair creation through Terraform can ensure standardized and consistent keypair configuration, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create KPS keypairs.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [KPS Keypair Resource (huaweicloud_kps_keypair)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create KPS Keypair Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a KPS keypair resource:

```hcl
variable "keypair_name" {
  description = "The name of the KPS keypair"
  type        = string
}

variable "keypair_scope" {
  description = "The scope of the KPS keypair"
  type        = string
  default     = "user"
}

variable "keypair_user_id" {
  description = "The user ID to which the KPS keypair belongs"
  type        = string
  default     = ""
}

variable "keypair_encryption_type" {
  description = "The encryption mode of the KPS keypair"
  type        = string
  default     = "kms"
}

variable "kms_key_id" {
  description = "The ID of the KMS key"
  type        = string
  default     = ""
}

variable "kms_key_name" {
  description = "The name of the KMS key"
  type        = string
  default     = ""

  validation {
    condition     = var.keypair_encryption_type != "kms" || (var.kms_key_id != "" || var.kms_key_name != "")
    error_message = "At least one of kms_key_id and kms_key_name must be provided when keypair_encryption_type set to kms"
  }
}

variable "keypair_description" {
  description = "The description of the KPS keypair"
  type        = string
  default     = ""
}

# Create KPS keypair resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_kps_keypair" "test" {
  name            = var.keypair_name
  scope           = var.keypair_scope
  user_id         = var.keypair_user_id != "" ? var.keypair_user_id : null
  encryption_type = var.keypair_encryption_type
  kms_key_id      = var.kms_key_id != "" ? var.kms_key_id : null
  kms_key_name    = var.kms_key_name != "" ? var.kms_key_name : null
  description     = var.keypair_description
}
```

**Parameter Description**:
- **name**: The keypair name, assigned by referencing the input variable keypair_name
- **scope**: The keypair scope, assigned by referencing the input variable keypair_scope, default value is "user" (user-level)
- **user_id**: The user ID to which the keypair belongs, assigned by referencing the input variable keypair_user_id, optional parameter, default value is null
- **encryption_type**: The keypair encryption mode, assigned by referencing the input variable keypair_encryption_type, default value is "kms" (use KMS encryption), valid values: default (default encryption), kms (KMS encryption)
- **kms_key_id**: The KMS key ID, assigned by referencing the input variable kms_key_id, when encryption_type is "kms", at least one of kms_key_id and kms_key_name must be provided, optional parameter, default value is null
- **kms_key_name**: The KMS key name, assigned by referencing the input variable kms_key_name, when encryption_type is "kms", at least one of kms_key_id and kms_key_name must be provided, optional parameter, default value is null
- **description**: The keypair description, assigned by referencing the input variable keypair_description, optional parameter, default value is empty string

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# KPS Keypair Configuration
keypair_name        = "tf_test_keypair"
kms_key_id          = "your_kms_key_id"
keypair_description = "This is a KPS keypair created by Terraform"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially `kms_key_id` needs to be replaced with the actual KMS key ID (when encryption_type is "kms")
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="keypair_name=my_keypair" -var="kms_key_id=my_kms_key_id"`
2. Environment variables: `export TF_VAR_keypair_name=my_keypair` and `export TF_VAR_kms_key_id=my_kms_key_id`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values. When encryption_type is set to "kms", at least one of kms_key_id and kms_key_name must be provided.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a KPS keypair:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the KPS keypair
4. Run `terraform show` to view the details of the created KPS keypair

> Note: After the KPS keypair is created, the public key can be injected into ECS instances to achieve passwordless login. Keypairs support encryption using KMS keys, providing higher security. The keypair creation process takes about 5 minutes. Please ensure that private key information is properly kept and do not commit private keys to version control systems.

## Reference Information

- [Huawei Cloud DEW Product Documentation](https://support.huaweicloud.com/dew/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Keypair](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dew/kps-keypair)
