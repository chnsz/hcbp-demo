# Deploy Attach Volume

## Application Scenario

Bare Metal Server (BMS) volume attachment is an important storage expansion function provided by the BMS service. When you need to expand the storage capacity of a BMS instance, you can increase the instance's storage space by attaching a cloud disk. Automating volume attachment operations through Terraform can ensure standardized and consistent storage resource management, improving operational efficiency. This best practice will introduce how to use Terraform to automatically attach a cloud disk to a BMS instance.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [BMS Instance Volume Attachment Resource (huaweicloud_bms_volume_attach)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/bms_volume_attach)

### Resource/Data Source Dependencies

```
huaweicloud_bms_volume_attach
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create BMS Instance Volume Attachment Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a BMS instance volume attachment resource:

```hcl
variable "server_id" {
  description = "The BMS instance ID"
  type        = string
}

variable "volume_id" {
  description = "The ID of the disk to be attached to a BMS instance"
  type        = string
}

variable "device" {
  description = "The mount point"
  type        = string
  default     = null
}

# Create BMS instance volume attachment resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_bms_volume_attach" "test" {
  server_id = var.server_id
  volume_id = var.volume_id
  device    = var.device
}
```

**Parameter Description**:
- **server_id**: The BMS instance ID, assigned by referencing the input variable server_id
- **volume_id**: The ID of the cloud disk to be attached to the BMS instance, assigned by referencing the input variable volume_id
- **device**: The mount point, such as /dev/sda, /dev/sdb, etc., assigned by referencing the input variable device, default value is null (system automatically assigns)

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, the resource uses input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# BMS Instance Volume Attachment Configuration
server_id = "your_bms_server_id"
volume_id = "your_evs_volume_id"
device    = "/dev/sdb"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="server_id=your-server-id" -var="volume_id=your-volume-id" -var="device=/dev/sdb"`
2. Environment variables: `export TF_VAR_server_id=your-server-id` and `export TF_VAR_volume_id=your-volume-id` and `export TF_VAR_device=/dev/sdb`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to attach a cloud disk to a BMS instance:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start attaching the cloud disk to the BMS instance
4. Run `terraform show` to view the details of the created BMS instance volume attachment resource

> Note: Destroying this resource will detach the volume from the BMS instance.

## Reference Information

- [Huawei Cloud BMS Product Documentation](https://support.huaweicloud.com/bms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Attach Volume](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/bms/volume-attach)
