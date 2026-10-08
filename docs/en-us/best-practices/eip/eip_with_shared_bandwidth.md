# Deploy EIP on Shared Bandwidth

## Application Scenario

Elastic IP (EIP) is an independently applicable and bindable public IP address resource provided by Huawei Cloud, offering cloud resources the ability to access the public network and be accessed from it. Shared bandwidth allows multiple EIPs to be added to the same bandwidth resource, enabling bandwidth reuse and sharing, thereby reducing public bandwidth costs and improving bandwidth utilization.

This best practice will introduce how to use Terraform to create a shared bandwidth and directly attach an Elastic IP to it during EIP creation, so that the EIP uses the shared bandwidth's public egress capability from the moment it is created. It is suitable for networking scenarios where public bandwidth needs to be planned and reused across multiple public IPs.

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

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

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
- **name**: Assigned by referencing the input variable bandwidth_name, specifying the name of the shared bandwidth
- **size**: Assigned by referencing the input variable bandwidth_size, specifying the size of the shared bandwidth in Mbit/s
- **charge_mode**: Assigned by referencing the input variable bandwidth_charge_mode, specifying the charge mode of the shared bandwidth
- **bandwidth_type**: Assigned by referencing the input variable bandwidth_type, specifying the type of the shared bandwidth
- **public_border_group**: Assigned by referencing the input variable bandwidth_public_border_group, specifying the border group of the public IP

### 3. Create an Elastic IP Attached to the Shared Bandwidth

Add the following script to the TF file (such as main.tf) to create an Elastic IP and directly attach it to the shared bandwidth created in the previous step through `share_type = "WHOLE"` and the shared bandwidth ID in its `bandwidth` block:

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
- **publicip.type**: Assigned by referencing the input variable eip_type, specifying the type of the Elastic IP
- **bandwidth.share_type**: Set to `WHOLE`, indicating that the Elastic IP uses shared bandwidth
- **bandwidth.id**: Assigned by referencing the ID of the shared bandwidth resource `huaweicloud_vpc_bandwidth.test`, attaching the Elastic IP to the specified shared bandwidth
- **description**: Assigned by referencing the input variable eip_description, specifying the description of the Elastic IP
- **tags**: Assigned by referencing the input variable eip_tags, specifying the tags of the Elastic IP

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Shared bandwidth name
bandwidth_name = "tf_test_shared_bandwidth"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the shared bandwidth and the Elastic IP attached to the shared bandwidth
4. Run `terraform show` to view the created shared bandwidth and the Elastic IP attached to the shared bandwidth

## Reference Information

- [Huawei Cloud Elastic IP Product Documentation](https://support.huaweicloud.com/eip/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EIP Shared Bandwidth](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eip/eip-with-shared-bandwidth)
