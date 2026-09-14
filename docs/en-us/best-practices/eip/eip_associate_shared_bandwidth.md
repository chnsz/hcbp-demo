# Deploy EIP Bound to Shared Bandwidth

## Application Scenario

Elastic IP (EIP) is an independently applicable and bindable public IP address resource provided by Huawei Cloud, supporting bandwidth-based or traffic-based billing. Shared Bandwidth allows multiple EIPs to be added to the same bandwidth resource, enabling bandwidth reuse and sharing, thereby reducing public bandwidth costs and improving bandwidth utilization.

This best practice will introduce how to use Terraform to automatically create a shared bandwidth and a dedicated EIP, and add the EIP to the shared bandwidth through an association resource, including shared bandwidth creation, EIP creation, and EIP-to-shared-bandwidth association management.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Shared Bandwidth (huaweicloud_vpc_bandwidth)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_bandwidth)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [EIP Bandwidth Associate (huaweicloud_eip_bandwidth_associate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eip_bandwidth_associate)

### Resource/Data Source Dependencies

```
huaweicloud_vpc_bandwidth
    └── huaweicloud_eip_bandwidth_associate

huaweicloud_vpc_eip
    └── huaweicloud_eip_bandwidth_associate
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Shared Bandwidth

Add the following script in the TF file (such as main.tf) to create a shared bandwidth:

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
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, and passed as null when the value is an empty string
- **name**: Assigned by referencing the input variable bandwidth_name
- **size**: Assigned by referencing the input variable bandwidth_size, indicating the size of the shared bandwidth in Mbit/s
- **charge_mode**: Assigned by referencing the input variable bandwidth_charge_mode, indicating the charge mode of the shared bandwidth
- **bandwidth_type**: Assigned by referencing the input variable bandwidth_type, indicating the bandwidth type, which must be set to share for shared bandwidth
- **public_border_group**: Assigned by referencing the input variable bandwidth_public_border_group, indicating the public border group

### 3. Create an Elastic IP

Add the following script in the TF file (such as main.tf) to create an elastic IP:

```hcl
# Create an elastic IP resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "eip_bandwidth_name" {
  description = "The name of the dedicated EIP bandwidth"
  type        = string
}

variable "eip_bandwidth_size" {
  description = "The size of the dedicated EIP bandwidth in Mbit/s"
  type        = number
  default     = 5
}

variable "eip_bandwidth_charge_mode" {
  description = "The charge mode of the dedicated EIP bandwidth"
  type        = string
  default     = "traffic"
}

variable "eip_description" {
  description = "The description of the EIP"
  type        = string
  default     = ""
}

variable "eip_tags" {
  description = "The tags of the EIP"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_vpc_eip" "test" {
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.eip_bandwidth_name
    size        = var.eip_bandwidth_size
    share_type  = "PER"
    charge_mode = var.eip_bandwidth_charge_mode
  }

  description = var.eip_description
  tags        = var.eip_tags

  # After being added to the shared bandwidth, bandwidth.share_type will be automatically set to WHOLE, so the change needs to be ignored
  lifecycle {
    ignore_changes = [bandwidth]
  }
}
```

**Parameter Description**:
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, and passed as null when the value is an empty string
- **publicip.type**: Assigned by referencing the input variable eip_type, indicating the type of the elastic IP
- **bandwidth.name**: Assigned by referencing the input variable eip_bandwidth_name, indicating the name of the dedicated bandwidth
- **bandwidth.size**: Assigned by referencing the input variable eip_bandwidth_size, indicating the size of the dedicated bandwidth in Mbit/s
- **bandwidth.share_type**: Fixed to PER, indicating a dedicated bandwidth
- **bandwidth.charge_mode**: Assigned by referencing the input variable eip_bandwidth_charge_mode, indicating the charge mode of the dedicated bandwidth
- **description**: Assigned by referencing the input variable eip_description
- **tags**: Assigned by referencing the input variable eip_tags
- **lifecycle.ignore_changes**: Ignores changes to the bandwidth field to avoid continuous plan diffs after the EIP is added to the shared bandwidth

### 4. Bind the Elastic IP to the Shared Bandwidth

Add the following script in the TF file (such as main.tf) to bind the elastic IP to the shared bandwidth:

```hcl
# Create an EIP bandwidth associate resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_eip_bandwidth_associate" "test" {
  publicip_id           = huaweicloud_vpc_eip.test.id
  bandwidth_id          = huaweicloud_vpc_bandwidth.test.id
  bandwidth_charge_mode = var.eip_bandwidth_charge_mode
  bandwidth_size        = var.eip_bandwidth_size
  bandwidth_name        = var.eip_bandwidth_name
}
```

**Parameter Description**:
- **publicip_id**: Assigned by referencing huaweicloud_vpc_eip.test.id, indicating the ID of the elastic IP to be bound
- **bandwidth_id**: Assigned by referencing huaweicloud_vpc_bandwidth.test.id, indicating the ID of the target shared bandwidth
- **bandwidth_charge_mode**: Assigned by referencing the input variable eip_bandwidth_charge_mode, indicating the dedicated bandwidth charge mode restored after the EIP is detached from the shared bandwidth
- **bandwidth_size**: Assigned by referencing the input variable eip_bandwidth_size, indicating the dedicated bandwidth size restored after the EIP is detached from the shared bandwidth
- **bandwidth_name**: Assigned by referencing the input variable eip_bandwidth_name, indicating the dedicated bandwidth name restored after the EIP is detached from the shared bandwidth

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
bandwidth_name     = "tf_test_shared_bandwidth"
eip_bandwidth_name = "tf_test_eip_bandwidth"
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

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the shared bandwidth and elastic IP, and bind the EIP to the shared bandwidth
4. Run `terraform show` to view the created shared bandwidth and elastic IP

## Reference Information

- [Huawei Cloud Elastic IP Product Documentation](https://support.huaweicloud.com/eip/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EIP Shared Bandwidth Association](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eip/eip-associate-shared-bandwidth)
