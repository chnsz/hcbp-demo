# Deploy EIP Bound to Shared Bandwidth

## Application Scenario

Shared bandwidth is a bandwidth reuse capability provided by Huawei Cloud, allowing multiple Elastic IP (EIP) addresses to be added to the same bandwidth resource, so that bandwidth can be shared and reused. This reduces public bandwidth costs and improves bandwidth utilization. In real-world scenarios, when multiple cloud servers or cloud resources need to access the public network, purchasing bandwidth separately for each EIP often leads to idle bandwidth and wasted cost, while shared bandwidth bills the overall peak uniformly, which is more economical and efficient.

This best practice will introduce how to use Terraform to automatically deploy a shared bandwidth and create an Elastic IP on it, implementing EIP-to-shared-bandwidth association management.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Shared Bandwidth (huaweicloud_vpc_bandwidth)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_bandwidth)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)

### Resource/Data Source Dependencies

```
huaweicloud_vpc_bandwidth
    └── huaweicloud_vpc_eip
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration details.

### 2. Create a Shared Bandwidth

Add the following script to the TF file (such as main.tf) to create a shared bandwidth:

```hcl
# Create a shared bandwidth resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
}

variable "bandwidth_name" {
  description = "The name of the shared bandwidth"
  type        = string
}

variable "bandwidth_size" {
  description = "The size of the shared bandwidth in Mbit/s"
  type        = number
  default     = 5
}

variable "bandwidth_charge_mode" {
  description = "The charge mode of the shared bandwidth"
  type        = string
  default     = "bandwidth"
}

variable "bandwidth_type" {
  description = "The type of the bandwidth"
  type        = string
  default     = "share"
}

variable "bandwidth_public_border_group" {
  description = "The border group of the public IP"
  type        = string
  default     = "center"
}

resource "huaweicloud_vpc_bandwidth" "test" {
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  name                  = var.bandwidth_name
  size                  = var.bandwidth_size
  charge_mode           = var.bandwidth_charge_mode
  bandwidth_type        = var.bandwidth_type
  public_border_group   = var.bandwidth_public_border_group
}
```

**Parameter Description**:
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; when it is an empty string, null is used to indicate that no enterprise project is specified
- **name**: Assigned by referencing the input variable bandwidth_name, used to specify the name of the shared bandwidth
- **size**: Assigned by referencing the input variable bandwidth_size, used to specify the size of the shared bandwidth in Mbit/s
- **charge_mode**: Assigned by referencing the input variable bandwidth_charge_mode, used to specify the charge mode of the shared bandwidth
- **bandwidth_type**: Assigned by referencing the input variable bandwidth_type, used to specify the type of the shared bandwidth
- **public_border_group**: Assigned by referencing the input variable bandwidth_public_border_group, used to specify the border group of the public IP

### 3. Create an Elastic IP and Bind It to the Shared Bandwidth

Add the following script to the TF file (such as main.tf) to create an Elastic IP and bind it to the shared bandwidth created in the previous step:

```hcl
# Create an Elastic IP resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "eip_description" {
  description = "The description of the EIP"
  type        = string
  default     = ""
}

variable "eip_tags" {
  description = "The tags of the EIP"
  type        = map(string)
  default     = null
}

resource "huaweicloud_vpc_eip" "test" {
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  publicip {
    type = var.eip_type
  }

  bandwidth {
    share_type = "WHOLE"
    id         = huaweicloud_vpc_bandwidth.test.id
  }

  description = var.eip_description
  tags        = var.eip_tags
}
```

**Parameter Description**:
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; when it is an empty string, null is used to indicate that no enterprise project is specified
- **publicip.type**: Assigned by referencing the input variable eip_type, used to specify the type of the Elastic IP
- **bandwidth.share_type**: Set to WHOLE, indicating that the EIP uses shared bandwidth
- **bandwidth.id**: References the ID of the shared bandwidth created in the previous step, implementing the association between the EIP and the shared bandwidth
- **description**: Assigned by referencing the input variable eip_description, used to specify the description of the Elastic IP
- **tags**: Assigned by referencing the input variable eip_tags, used to specify the tags of the Elastic IP

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Shared bandwidth configuration
bandwidth_name = "tf_test_shared_bandwidth"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="bandwidth_name=my-bandwidth"`
2. Environment variables: `export TF_VAR_bandwidth_name=my-bandwidth`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the shared bandwidth and Elastic IP
4. Run `terraform show` to view the created shared bandwidth and Elastic IP

## Reference Information

- [Huawei Cloud Elastic IP Product Documentation](https://support.huaweicloud.com/eip/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EIP Shared Bandwidth](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eip/eip-with-shared-bandwidth)
