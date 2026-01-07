# Deploy Reset Instance Password

## Application Scenario

Bare Metal Server (BMS) instance password reset is an important operation and maintenance function provided by the BMS service. When you forget the instance password or need to regularly change the password, you can quickly recover or update the instance login password through the password reset function. Automating password reset operations through Terraform can ensure standardized and secure password management, avoiding security risks that may arise from manual operations. This best practice will introduce how to use Terraform to automatically reset the password of a BMS instance.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [BMS Instance Password Reset Resource (huaweicloud_bms_instance_password_reset)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/bms_instance_password_reset)

### Resource/Data Source Dependencies

```
huaweicloud_bms_instance_password_reset
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create BMS Instance Password Reset Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a BMS instance password reset resource:

```hcl
variable "bms_instance_id" {
  description = "The ID of the BMS instance"
  type        = string
}

variable "bms_instance_new_password" {
  description = "The new password of the BMS instance"
  type        = string
  sensitive   = true
}

# Create BMS instance password reset resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_bms_instance_password_reset" "test" {
  server_id    = var.bms_instance_id
  new_password = var.bms_instance_new_password
}
```

**Parameter Description**:
- **server_id**: The BMS instance ID, assigned by referencing the input variable bms_instance_id
- **new_password**: The new password of the BMS instance, assigned by referencing the input variable bms_instance_new_password, which is marked as sensitive information

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, the resource uses input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# BMS Instance Password Reset Configuration
bms_instance_id           = "your_bms_instance_id"
bms_instance_new_password = "your_new_password"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="bms_instance_id=your-instance-id" -var="bms_instance_new_password=your-password"`
2. Environment variables: `export TF_VAR_bms_instance_id=your-instance-id` and `export TF_VAR_bms_instance_new_password=your-password`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to reset the BMS instance password:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start resetting the BMS instance password
4. Run `terraform show` to view the details of the created BMS instance password reset resource

> Note: Destroying this module will not clear the corresponding request record, but will only remove the resource information from the tf state file.

## Reference Information

- [Huawei Cloud BMS Product Documentation](https://support.huaweicloud.com/bms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Reset Instance Password](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/bms/bms-reset-password)
