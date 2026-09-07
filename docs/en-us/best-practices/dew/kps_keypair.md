# Deploy Keypair

## Application Scenario

Key Pair Service (KPS) is a service provided by Huawei Cloud for managing SSH key pairs, helping you securely and conveniently manage the login credentials of ECS instances. By using Terraform, you can automatically create and manage KPS key pairs and inject the public key into ECS instances to achieve passwordless login, improving security and operational efficiency.

This best practice will introduce how to use Terraform to create a KPS key pair on Huawei Cloud, and configure its encryption mode, owning user, description, and other information. Through this practice, you can learn how to efficiently manage key pair resources on the cloud using Infrastructure as Code (IaC).

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Key Pair Management Service Keypair (huaweicloud_kps_keypair)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)

### Resource/Data Source Dependencies

```
huaweicloud_kps_keypair
```

## Operation Steps

### 1. Script Preparation

Prepare the TF files (such as main.tf) required for writing the best practice script in the specified workspace, and ensure that they (or other TF files in the same level directory) contain the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a KPS Keypair

Add the following script to the TF file (such as main.tf):

```hcl
# Create a KPS keypair in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
    error_message = "At least one of kms_key_id and kms_key_name must be provided when keypair_encryption_type set to **kms**"
  }
}

variable "keypair_description" {
  description = "The description of the KPS keypair"
  type        = string
  default     = ""
}

resource "huaweicloud_kps_keypair" "test" {
  name            = var.keypair_name
  scope           = var.keypair_scope
  user_id         = var.keypair_user_id
  encryption_type = var.keypair_encryption_type
  kms_key_id      = var.kms_key_id
  kms_key_name    = var.kms_key_name
  description     = var.keypair_description
}
```

**Parameter Description**:

- **name**: Assigned by referencing the input variable keypair_name, used to specify the name of the key pair.
- **scope**: Assigned by referencing the input variable keypair_scope, used to specify the scope of the key pair, defaults to `user`.
- **user_id**: Assigned by referencing the input variable keypair_user_id, used to specify the user ID to which the key pair belongs.
- **encryption_type**: Assigned by referencing the input variable keypair_encryption_type, used to specify the encryption mode of the key pair, defaults to `kms`.
- **kms_key_id**: Assigned by referencing the input variable kms_key_id, used to specify the ID of the KMS key. When the encryption mode is `kms`, at least one of kms_key_id and kms_key_name must be provided.
- **kms_key_name**: Assigned by referencing the input variable kms_key_name, used to specify the name of the KMS key. When the encryption mode is `kms`, at least one of kms_key_id and kms_key_name must be provided.
- **description**: Assigned by referencing the input variable keypair_description, used to specify the description of the key pair.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
keypair_name        = "tf_test_keypair"
kms_key_id          = "your_kms_key_id"
keypair_description = "This is a KPS keypair created by Terraform"
```

**Usage**:

1. Save the above content as the `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; other names need to add `.auto` before `tfvars`, such as `variables.auto.tfvars`)
2. Modify the parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="keypair_name=my-keypair"`
2. Environment variables: `export TF_VAR_keypair_name=my-keypair`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the KPS keypair
4. Run `terraform show` to view the created KPS keypair

## Reference Information

- [Huawei Cloud Data Encryption Workshop Product Documentation](https://support.huaweicloud.com/dew/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DEW Keypair](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dew/kps-keypair)
