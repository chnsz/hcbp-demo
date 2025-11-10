# Deploy VPN Gateway

## Application Scenario

Virtual Private Network (VPN) is a secure and reliable network connection service provided by Huawei Cloud, supporting the establishment of encrypted IPsec VPN connections between VPCs and local networks, enabling interconnection between cloud resources and local data centers. VPN gateway is the core component of VPN service, providing highly available VPN connection capabilities, supporting dual EIP deployment to ensure VPN connection stability and reliability.

This best practice will introduce how to use Terraform to automatically deploy a VPN gateway, including VPN gateway availability zone query, VPC, subnet, EIP, and VPN gateway creation. Through VPN gateway, you can achieve secure connections between cloud VPCs and local networks, building hybrid cloud architectures.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [VPN Gateway Availability Zones Query Data Source (data.huaweicloud_vpn_gateway_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/vpn_gateway_availability_zones)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [EIP Resource (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [VPN Gateway Resource (huaweicloud_vpn_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpn_gateway)

### Resource/Data Source Dependencies

```
data.huaweicloud_vpn_gateway_availability_zones
    └── huaweicloud_vpn_gateway

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_vpn_gateway

huaweicloud_vpc_eip
    └── huaweicloud_vpn_gateway
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration introduction.

### 2. Query VPN Gateway Availability Zone Information Through Data Source

Add the following script to the TF file (such as main.tf) to instruct Terraform to perform a data source query, the query results are used to create VPN gateway resources:

```hcl
variable "vpn_gateway_flavor" {
  description = "The flavor of the VPN gateway"
  type        = string
  default     = "professional1"
}

variable "vpn_gateway_attachment_type" {
  description = "The attachment type of the VPC"
  type        = string
  default     = "vpc"
}

# Query all VPN gateway availability zone information that meets the conditions in the specified region (defaults to the region specified in the current provider block when the region parameter is omitted) for creating VPN gateway resources
data "huaweicloud_vpn_gateway_availability_zones" "test" {
  flavor          = var.vpn_gateway_flavor
  attachment_type = var.vpn_gateway_attachment_type
}
```

**Parameter Description**:
- **flavor**: The flavor of the VPN gateway, assigned by referencing the input variable `vpn_gateway_flavor`, default is "professional1" indicating Professional Edition Type 1
- **attachment_type**: The attachment type of the VPC, assigned by referencing the input variable `vpn_gateway_attachment_type`, default is "vpc" indicating VPC type

### 3. Create VPC Resource

Add the following script to the TF file to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

# Create a VPC resource in the specified region (defaults to the region specified in the current provider block when the region parameter is omitted) for deploying VPN gateway
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: The name of the VPC, assigned by referencing the input variable `vpc_name`
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable `vpc_cidr`, default is "192.168.0.0/16" block

### 4. Create VPC Subnet Resource

Add the following script to the TF file to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
}

# Create a VPC subnet resource in the specified region (defaults to the region specified in the current provider block when the region parameter is omitted) for deploying VPN gateway
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **vpc_id**: The VPC ID to which the subnet belongs, referencing the ID of the VPC resource created earlier
- **name**: The name of the subnet, assigned by referencing the input variable `subnet_name`
- **cidr**: The CIDR block of the subnet, if subnet CIDR is specified, use that value, otherwise use the cidrsubnet function to divide a subnet segment from the VPC's CIDR block
- **gateway_ip**: The gateway IP of the subnet, if gateway IP is specified, use that value, otherwise use the cidrhost function to get the first IP address from the subnet segment as the gateway IP

### 5. Create EIP Resources

Add the following script to the TF file to instruct Terraform to create EIP resources (VPN gateway requires two EIPs for high availability):

```hcl
variable "eip_type" {
  description = "The EIP type"
  type        = string
  default     = "5_bgp"
}

variable "bandwidth_name" {
  description = "The bandwidth name"
  type        = string
}

variable "bandwidth_size" {
  description = "The bandwidth size"
  type        = number
  default     = 8
}

variable "bandwidth_share_type" {
  description = "The bandwidth share type"
  type        = string
  default     = "PER"
}

variable "bandwidth_charge_mode" {
  description = "The bandwidth charge mode"
  type        = string
  default     = "traffic"
}

# Create EIP resources in the specified region (defaults to the region specified in the current provider block when the region parameter is omitted) for deploying VPN gateway
resource "huaweicloud_vpc_eip" "test" {
  count = 2

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = "${var.bandwidth_name}-${count.index}"
    size        = var.bandwidth_size
    share_type  = var.bandwidth_share_type
    charge_mode = var.bandwidth_charge_mode
  }
}
```

**Parameter Description**:
- **count**: The number of resource creations, set to 2 to create two EIP resources for VPN gateway high availability deployment
- **publicip**: Public IP configuration block
  - **type**: EIP type, assigned by referencing the input variable `eip_type`, default is "5_bgp" indicating full dynamic BGP
- **bandwidth**: Bandwidth configuration block
  - **name**: The name of the bandwidth, assigned by referencing the input variable `bandwidth_name` and count index to ensure each EIP's bandwidth name is unique
  - **size**: The size of the bandwidth (Mbps), assigned by referencing the input variable `bandwidth_size`, default is 8Mbps
  - **share_type**: The share type of the bandwidth, assigned by referencing the input variable `bandwidth_share_type`, default is "PER" indicating dedicated
  - **charge_mode**: The charge mode of the bandwidth, assigned by referencing the input variable `bandwidth_charge_mode`, default is "traffic" indicating pay-per-traffic

> Note: VPN gateway requires two EIPs for high availability deployment to ensure VPN connection stability and reliability.

### 6. Create VPN Gateway Resource

Add the following script to the TF file to instruct Terraform to create a VPN gateway resource:

```hcl
variable "vpn_gateway_name" {
  description = "The name of the VPN gateway"
  type        = string
}

variable "vpn_gateway_delete_eip_on_termination" {
  description = "Whether to delete the EIP when the VPN gateway is deleted."
  type        = bool
  default     = false
}

# Create a VPN gateway resource in the specified region (defaults to the region specified in the current provider block when the region parameter is omitted)
resource "huaweicloud_vpn_gateway" "test" {
  name               = var.vpn_gateway_name
  vpc_id             = huaweicloud_vpc.test.id
  local_subnets      = [huaweicloud_vpc_subnet.test.cidr]
  connect_subnet     = huaweicloud_vpc_subnet.test.id
  availability_zones = [
    try(data.huaweicloud_vpn_gateway_availability_zones.test.names[0], "default_value"),
    try(data.huaweicloud_vpn_gateway_availability_zones.test.names[1], "default_value")
  ]

  eip1 {
    id = huaweicloud_vpc_eip.test[0].id
  }

  eip2 {
    id = huaweicloud_vpc_eip.test[1].id
  }

  delete_eip_on_termination = var.vpn_gateway_delete_eip_on_termination
}
```

**Parameter Description**:
- **name**: The name of the VPN gateway, assigned by referencing the input variable `vpn_gateway_name`
- **vpc_id**: The VPC ID to which the VPN gateway belongs, referencing the ID of the VPC resource created earlier
- **local_subnets**: The local subnet list of the VPN gateway, referencing the CIDR block of the subnet resource
- **connect_subnet**: The connection subnet ID of the VPN gateway, referencing the ID of the subnet resource
- **availability_zones**: The availability zone list where the VPN gateway is located, assigned based on the return result of the VPN gateway availability zone query data source, using the first two availability zones for high availability
- **eip1**: First EIP configuration block
  - **id**: The ID of the first EIP, referencing the ID of the first EIP resource
- **eip2**: Second EIP configuration block
  - **id**: The ID of the second EIP, referencing the ID of the second EIP resource
- **delete_eip_on_termination**: Whether to delete EIPs when the VPN gateway is deleted, assigned by referencing the input variable `vpn_gateway_delete_eip_on_termination`, default is false indicating not to delete EIPs

> Note: VPN gateway requires two EIPs configured for high availability deployment, with two EIPs deployed in different availability zones to ensure VPN connection stability and reliability.

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory, with example content as follows:

```hcl
# Network resource configuration
vpc_name         = "tf_test_vpc"
subnet_name      = "tf_test_subnet"
bandwidth_name   = "tf_test_bandwidth"

# VPN gateway configuration
vpn_gateway_name = "tf_test_gateway"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content in this `tfvars` file when executing terraform commands, other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="vpn_gateway_name=my-gateway"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the VPN gateway
4. Run `terraform show` to view the created VPN gateway details

## Reference Information

- [Huawei Cloud VPN Product Documentation](https://support.huaweicloud.com/intl/en-us/vpn/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For VPN Gateway](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpn/gateway)
