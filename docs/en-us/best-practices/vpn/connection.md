# Deploy Connection

## Application Scenario

Virtual Private Network (VPN) is a secure and reliable network connection service provided by Huawei Cloud, supporting the establishment of encrypted IPsec VPN connections between VPCs and local networks, enabling interconnection between cloud resources and local data centers. VPN connection is a core function of VPN service, used to establish IPsec VPN connections between VPN gateways and customer gateways, achieving secure communication between cloud VPCs and local networks. Through VPN connections, encrypted tunnels can be established, ensuring data transmission security and stability, supporting multiple VPN types and configuration options, meeting network connection requirements for different scenarios. This best practice introduces how to use Terraform to automatically deploy VPN connections, including VPN gateway availability zone query, VPC, subnet, EIP, VPN gateway, customer gateway, and VPN connection creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [VPN Gateway Availability Zones Query Data Source (data.huaweicloud_vpn_gateway_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/vpn_gateway_availability_zones)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [EIP Resource (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [VPN Gateway Resource (huaweicloud_vpn_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpn_gateway)
- [Customer Gateway Resource (huaweicloud_vpn_customer_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpn_customer_gateway)
- [VPN Connection Resource (huaweicloud_vpn_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpn_connection)

### Resource/Data Source Dependencies

```text
data.huaweicloud_vpn_gateway_availability_zones
    └── huaweicloud_vpn_gateway
        └── huaweicloud_vpn_connection

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_vpn_gateway
            └── huaweicloud_vpn_connection

huaweicloud_vpc_eip
    └── huaweicloud_vpn_gateway
        └── huaweicloud_vpn_connection

huaweicloud_vpn_customer_gateway
    └── huaweicloud_vpn_connection
```

> Note: VPN connection depends on VPN gateway and customer gateway. VPN gateway depends on VPC, subnet, and EIP resources. VPN gateway availability zone query is used to obtain availability zone information available for VPN gateways.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Query VPN Gateway Availability Zones

Add the following script to the TF file (such as main.tf) to query VPN gateway availability zones:

```hcl
# Query VPN gateway availability zones data source
data "huaweicloud_vpn_gateway_availability_zones" "test" {
  flavor          = var.vpn_gateway_az_flavor
  attachment_type = var.vpn_gateway_az_attachment_type
}
```

**Parameter Description**:
- **flavor**: VPN gateway flavor name, assigned by referencing the input variable `vpn_gateway_az_flavor`
- **attachment_type**: VPN gateway attachment type, assigned by referencing the input variable `vpn_gateway_az_attachment_type`

### 3. Create VPC and Subnet

Add the following script to the TF file (such as main.tf) to create VPC and subnet:

```hcl
# Create VPC resource
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

# Create VPC subnet resource
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable `vpc_name`
- **cidr**: VPC CIDR block, assigned by referencing the input variable `vpc_cidr`
- **vpc_id**: VPC ID to which the subnet belongs, assigned by referencing the VPC resource ID
- **cidr**: Subnet CIDR block, automatically calculated if the input variable is empty, otherwise uses the input variable value
- **gateway_ip**: Subnet gateway IP address, automatically calculated if the input variable is empty, otherwise uses the input variable value

### 4. Create EIP

Add the following script to the TF file (such as main.tf) to create EIP:

```hcl
# Create EIP resources (need to create 2 EIPs for VPN gateway dual EIP deployment)
resource "huaweicloud_vpc_eip" "test" {
  count = 2

  dynamic "publicip" {
    for_each = var.vpc_eip_public_ip

    content {
      type = publicip.value.type
    }
  }

  dynamic "bandwidth" {
    for_each = var.vpc_eip_bandwidth

    content {
      name        = bandwidth.value.name
      size        = bandwidth.value.size
      share_type  = bandwidth.value.share_type
      charge_mode = bandwidth.value.charge_mode
    }
  }
}
```

**Parameter Description**:
- **publicip**: Public IP configuration, creates public IP configuration through dynamic block `dynamic "publicip"` based on input variable `vpc_eip_public_ip`
  - **type**: Public IP type, assigned by referencing the `type` in the input variable
- **bandwidth**: Bandwidth configuration, creates bandwidth configuration through dynamic block `dynamic "bandwidth"` based on input variable `vpc_eip_bandwidth`
  - **name**: Bandwidth name, assigned by referencing the `name` in the input variable
  - **size**: Bandwidth size, assigned by referencing the `size` in the input variable
  - **share_type**: Bandwidth sharing type, assigned by referencing the `share_type` in the input variable
  - **charge_mode**: Bandwidth billing mode, assigned by referencing the `charge_mode` in the input variable

> Note: VPN gateway requires 2 EIPs for dual EIP deployment to ensure high availability of VPN connections.

### 5. Create VPN Gateway

Add the following script to the TF file (such as main.tf) to create VPN gateway:

```hcl
# Create VPN gateway resource
resource "huaweicloud_vpn_gateway" "test" {
  name               = var.vpn_gateway_name
  vpc_id             = huaweicloud_vpc.test.id
  local_subnets      = [huaweicloud_vpc_subnet.test.cidr]
  connect_subnet     = huaweicloud_vpc_subnet.test.id
  availability_zones = [
    try(data.huaweicloud_vpn_gateway_availability_zones.test.names[0], null),
    try(data.huaweicloud_vpn_gateway_availability_zones.test.names[1], null)
  ]

  eip1 {
    id = huaweicloud_vpc_eip.test[0].id
  }

  eip2 {
    id = huaweicloud_vpc_eip.test[1].id
  }
}
```

**Parameter Description**:
- **name**: VPN gateway name, assigned by referencing the input variable `vpn_gateway_name`
- **vpc_id**: VPC ID to which the VPN gateway belongs, assigned by referencing the VPC resource ID
- **local_subnets**: Local subnet list of the VPN gateway, assigned by referencing the subnet resource CIDR
- **connect_subnet**: Connection subnet ID of the VPN gateway, assigned by referencing the subnet resource ID
- **availability_zones**: Availability zone list of the VPN gateway, assigned by referencing the VPN gateway availability zone query data source results
- **eip1**: First EIP configuration of the VPN gateway, assigned by referencing the first EIP resource ID
- **eip2**: Second EIP configuration of the VPN gateway, assigned by referencing the second EIP resource ID

### 6. Create Customer Gateway

Add the following script to the TF file (such as main.tf) to create customer gateway:

```hcl
# Create customer gateway resource
resource "huaweicloud_vpn_customer_gateway" "test" {
  name     = var.vpn_customer_gateway_name
  id_value = var.vpn_customer_gateway_id_value
}
```

**Parameter Description**:
- **name**: Customer gateway name, assigned by referencing the input variable `vpn_customer_gateway_name`
- **id_value**: Customer gateway identifier (usually a public IP address), assigned by referencing the input variable `vpn_customer_gateway_id_value`

### 7. Create VPN Connection

Add the following script to the TF file (such as main.tf) to create VPN connection:

```hcl
# Create VPN connection resource
resource "huaweicloud_vpn_connection" "test" {
  name                = var.vpn_connection_name
  gateway_id          = huaweicloud_vpn_gateway.test.id
  gateway_ip          = huaweicloud_vpn_gateway.test.master_eip[0].id
  customer_gateway_id = huaweicloud_vpn_customer_gateway.test.id
  peer_subnets        = var.vpn_connection_peer_subnets
  vpn_type            = var.vpn_connection_vpn_type
  psk                 = var.vpn_connection_psk
  enable_nqa          = var.vpn_connection_enable_nqa
}
```

**Parameter Description**:
- **name**: VPN connection name, assigned by referencing the input variable `vpn_connection_name`
- **gateway_id**: VPN gateway ID, assigned by referencing the VPN gateway resource ID
- **gateway_ip**: VPN gateway IP address, assigned by referencing the master EIP ID of the VPN gateway resource
- **customer_gateway_id**: Customer gateway ID, assigned by referencing the customer gateway resource ID
- **peer_subnets**: Peer subnet list, assigned by referencing the input variable `vpn_connection_peer_subnets`
- **vpn_type**: VPN connection type, assigned by referencing the input variable `vpn_connection_vpn_type`
- **psk**: Pre-shared key, assigned by referencing the input variable `vpn_connection_psk`
- **enable_nqa**: Whether to enable NQA detection, assigned by referencing the input variable `vpn_connection_enable_nqa`

> Note: VPN connection can only be created after VPN gateway and customer gateway are created. The pre-shared key (PSK) must be consistent with the peer gateway configuration to ensure the VPN connection can be established normally.

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# VPN gateway availability zone query configuration (Optional)
vpn_gateway_az_flavor          = "professional1"
vpn_gateway_az_attachment_type = "vpc"

# VPC and subnet configuration (Required)
vpc_name   = "tf_test_vpn_connection"
vpc_cidr   = "192.168.0.0/16"
subnet_name = "tf_test_vpn_connection"

# VPN gateway configuration (Required)
vpn_gateway_name = "tf_test_vpn_connection"

# Customer gateway configuration (Required)
vpn_customer_gateway_name  = "tf_test_vpn_connection"
vpn_customer_gateway_id_value = "8.8.8.8"

# VPN connection configuration (Required)
vpn_connection_name         = "tf_test_vpn_connection"
vpn_connection_peer_subnets = ["10.0.0.0/24"]
vpn_connection_vpn_type     = "ipsec"
vpn_connection_psk           = "test_psk_123456"
vpn_connection_enable_nqa    = true
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="vpc_name=tf_test_vpn_connection"`
2. Environment variables: `export TF_VAR_vpc_name=tf_test_vpn_connection`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating VPN connection and related resources
4. Run `terraform show` to view the created VPN connection

## Reference Information

- [Huawei Cloud VPN Product Documentation](https://support.huaweicloud.com/intl/en-us/vpn/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Connection](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpn/connection)
