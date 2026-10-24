# Deploy NAT Gateway Shared via VPC Peering

## Application Scenario

NAT Gateway is a network address translation service provided by Huawei Cloud, enabling cloud resources in a Virtual Private Cloud (VPC) to access the internet and be accessed from the internet. When an enterprise has multiple VPCs and wants to share a single set of public egress and inbound mappings, it can deploy a NAT gateway in one VPC and use VPC peering connections to steer traffic from other VPCs to that NAT gateway, reducing duplicate EIP and NAT gateway resources and lowering public access costs.

This best practice will introduce how to use Terraform to create a NAT gateway, EIPs, SNAT rules, and DNAT rules in VPC One, and use a VPC peering connection with route configurations so that ECS instances in VPC Other can access the internet through the NAT gateway in VPC One, while also supporting public access to ECS instances in both VPCs through DNAT rules.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Image (data.huaweicloud_images_image)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_image)
- [Compute Flavors (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### Resources

- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Elastic Cloud Server (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [VPC Peering Connection (huaweicloud_vpc_peering_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_peering_connection)
- [VPC Route (huaweicloud_vpc_route)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_route)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [NAT Gateway (huaweicloud_nat_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_gateway)
- [SNAT Rule (huaweicloud_nat_snat_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_snat_rule)
- [DNAT Rule (huaweicloud_nat_dnat_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_dnat_rule)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_compute_instance

data.huaweicloud_images_image
    └── huaweicloud_compute_instance

data.huaweicloud_compute_flavors
    └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
    └── huaweicloud_vpc_peering_connection
    └── huaweicloud_vpc_route
    └── huaweicloud_nat_gateway

huaweicloud_vpc_subnet
    └── huaweicloud_compute_instance
    └── huaweicloud_nat_gateway
    └── huaweicloud_nat_snat_rule

huaweicloud_vpc_peering_connection
    └── huaweicloud_vpc_route

huaweicloud_vpc_eip
    └── huaweicloud_nat_snat_rule
    └── huaweicloud_nat_dnat_rule

huaweicloud_nat_gateway
    └── huaweicloud_nat_snat_rule
    └── huaweicloud_nat_dnat_rule

huaweicloud_compute_instance
    └── huaweicloud_nat_dnat_rule
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the article [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones, Image, and Compute Flavors

Add the following script in the TF file (such as main.tf):

```hcl
# Query availability zones, image, and compute flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "image_name" {
  description = "The name of the image to use for ECS instances"
  type        = string
  default     = "Ubuntu 22.04 server 64bit"
}

variable "image_visibility" {
  description = "The visibility of the image"
  type        = string
  default     = "public"
}

variable "image_most_recent" {
  description = "If more than one result is returned, use the most recent image"
  type        = bool
  default     = true
}

variable "flavor_performance_type" {
  description = "The performance type of the flavor"
  type        = string
  default     = "normal"
}

variable "flavor_cpu_core_count" {
  description = "The number of CPU cores for the flavor"
  type        = number
  default     = 2
}

variable "flavor_memory_size" {
  description = "The memory size in GB for the flavor"
  type        = number
  default     = 4
}

data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_images_image" "test" {
  name        = var.image_name
  visibility  = var.image_visibility
  most_recent = var.image_most_recent
}

data "huaweicloud_compute_flavors" "test" {
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.flavor_performance_type
  cpu_core_count    = var.flavor_cpu_core_count
  memory_size       = var.flavor_memory_size
}
```

**Parameter Description**:
- **image_name**: Assigned by referencing the input variable image_name, used to specify the image name for ECS instances
- **image_visibility**: Assigned by referencing the input variable image_visibility, used to specify the image visibility
- **image_most_recent**: Assigned by referencing the input variable image_most_recent, used to specify whether to use the most recent image when multiple images are returned
- **flavor_performance_type**: Assigned by referencing the input variable flavor_performance_type, used to specify the performance type of the compute flavor
- **flavor_cpu_core_count**: Assigned by referencing the input variable flavor_cpu_core_count, used to specify the CPU core count of the compute flavor
- **flavor_memory_size**: Assigned by referencing the input variable flavor_memory_size, used to specify the memory size of the compute flavor

### 3. Create Security Group and Security Group Rules

Add the following script in the TF file (such as main.tf):

```hcl
# Create a security group and security group rules in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "secgroup_name" {
  description = "The name of the security group"
  type        = string
  default     = "secgroup-example"
}

variable "security_group_rules" {
  description = "List of security group rules"
  type        = list(object({
    direction        = string
    description      = string
    ethertype        = string
    protocol         = string
    ports            = optional(string)
    remote_ip_prefix = string
  }))
  default     = [
    {
      direction        = "ingress"
      description      = "Allow SSH access"
      ethertype        = "IPv4"
      protocol         = "tcp"
      ports            = "22"
      remote_ip_prefix = "0.0.0.0/0"
    },
    {
      direction        = "ingress"
      description      = "Allow HTTP and HTTPS access"
      ethertype        = "IPv4"
      protocol         = "tcp"
      ports            = "80,443"
      remote_ip_prefix = "0.0.0.0/0"
    },
    {
      direction        = "ingress"
      description      = "Allow ICMP access"
      ethertype        = "IPv4"
      protocol         = "icmp"
      ports            = null
      remote_ip_prefix = "0.0.0.0/0"
    }
  ]
}

resource "huaweicloud_networking_secgroup" "test" {
  name        = var.secgroup_name
  description = "Security group for ECS instances"
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = length(var.security_group_rules)

  direction         = var.security_group_rules[count.index].direction
  description       = var.security_group_rules[count.index].description
  ethertype         = var.security_group_rules[count.index].ethertype
  protocol          = var.security_group_rules[count.index].protocol
  ports             = var.security_group_rules[count.index].ports
  remote_ip_prefix  = var.security_group_rules[count.index].remote_ip_prefix
  security_group_id = huaweicloud_networking_secgroup.test.id
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable secgroup_name, used to specify the security group name
- **direction**: Assigned by referencing the direction in the input variable security_group_rules, used to specify the rule direction
- **description**: Assigned by referencing the description in the input variable security_group_rules, used to specify the rule description
- **ethertype**: Assigned by referencing the ethertype in the input variable security_group_rules, used to specify the IP protocol type
- **protocol**: Assigned by referencing the protocol in the input variable security_group_rules, used to specify the protocol type
- **ports**: Assigned by referencing the ports in the input variable security_group_rules, used to specify the port range
- **remote_ip_prefix**: Assigned by referencing the remote_ip_prefix in the input variable security_group_rules, used to specify the remote IP address range
- **security_group_id**: Assigned by referencing the ID of the security group resource huaweicloud_networking_secgroup.test

### 4. Create VPCs and Subnets

Add the following script in the TF file (such as main.tf):

```hcl
# Create VPCs and subnets in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpcs" {
  description = "List of VPC configurations"
  type        = list(object({
    vpc_name             = string
    vpc_cidr             = string
    subnet_name          = string
    subnet_cidr          = string
    subnet_gateway_ip    = optional(string)
    instance_name        = string
    ecs_flavor_id        = optional(string)
    ecs_image_id         = optional(string)
    ecs_system_disk_type = optional(string)
    ecs_system_disk_size = optional(number)
    ecs_admin_password   = optional(string)
    ecs_instance_tags    = optional(map(string))
  }))
  default     = [
    {
      vpc_name             = "vpc-one"
      vpc_cidr             = "192.168.0.0/16"
      subnet_name          = "subnet-one"
      subnet_cidr          = "192.168.1.0/24"
      subnet_gateway_ip    = "192.168.1.1"
      instance_name        = "instance-one"
      ecs_flavor_id        = null
      ecs_image_id         = null
      ecs_system_disk_type = null
      ecs_system_disk_size = null
      ecs_admin_password   = null
      ecs_instance_tags    = null
    },
    {
      vpc_name             = "vpc-other"
      vpc_cidr             = "10.0.0.0/16"
      subnet_name          = "subnet-other"
      subnet_cidr          = "10.0.2.0/24"
      subnet_gateway_ip    = "10.0.2.1"
      instance_name        = "instance-other"
      ecs_flavor_id        = null
      ecs_image_id         = null
      ecs_system_disk_type = null
      ecs_system_disk_size = null
      ecs_admin_password   = null
      ecs_instance_tags    = null
    }
  ]
}

resource "huaweicloud_vpc" "test" {
  count = length(var.vpcs)

  name = var.vpcs[count.index].vpc_name
  cidr = var.vpcs[count.index].vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  count = length(var.vpcs)

  vpc_id     = huaweicloud_vpc.test[count.index].id
  name       = var.vpcs[count.index].subnet_name
  cidr       = var.vpcs[count.index].subnet_cidr
  gateway_ip = var.vpcs[count.index].subnet_gateway_ip != null && var.vpcs[count.index].subnet_gateway_ip != "" ? var.vpcs[count.index].subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test[count.index].cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **name**: Assigned by referencing the vpc_name in the input variable vpcs, used to specify the VPC name
- **cidr**: Assigned by referencing the vpc_cidr in the input variable vpcs, used to specify the VPC CIDR block
- **vpc_id**: Assigned by referencing the ID of the VPC resource huaweicloud_vpc.test
- **name** (subnet): Assigned by referencing the subnet_name in the input variable vpcs, used to specify the subnet name
- **cidr** (subnet): Assigned by referencing the subnet_cidr in the input variable vpcs, used to specify the subnet CIDR block
- **gateway_ip**: Assigned by referencing the subnet_gateway_ip in the input variable vpcs; if not specified, the subnet gateway IP is automatically calculated

### 5. Create ECS Instances

Add the following script in the TF file (such as main.tf):

```hcl
# Create ECS instances in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "ecs_admin_password" {
  description = "The administrator password of ECS instances (used as fallback when not specified in vpcs)"
  type        = string
  sensitive   = true
}

resource "huaweicloud_compute_instance" "test" {
  count = length(var.vpcs)

  name               = var.vpcs[count.index].instance_name
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)
  flavor_id          = var.vpcs[count.index].ecs_flavor_id != null && var.vpcs[count.index].ecs_flavor_id != "" ? var.vpcs[count.index].ecs_flavor_id : try(data.huaweicloud_compute_flavors.test.ids[0], null)
  image_id           = var.vpcs[count.index].ecs_image_id != null && var.vpcs[count.index].ecs_image_id != "" ? var.vpcs[count.index].ecs_image_id : try(data.huaweicloud_images_image.test.id, null)
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  system_disk_type   = var.vpcs[count.index].ecs_system_disk_type != null ? var.vpcs[count.index].ecs_system_disk_type : "SSD"
  system_disk_size   = var.vpcs[count.index].ecs_system_disk_size != null ? var.vpcs[count.index].ecs_system_disk_size : 40

  network {
    uuid = huaweicloud_vpc_subnet.test[count.index].id
  }

  admin_pass = var.vpcs[count.index].ecs_admin_password != null && var.vpcs[count.index].ecs_admin_password != "" ? var.vpcs[count.index].ecs_admin_password : var.ecs_admin_password
  tags       = var.vpcs[count.index].ecs_instance_tags != null ? var.vpcs[count.index].ecs_instance_tags : {}

  depends_on = [
    huaweicloud_nat_gateway.test,
    huaweicloud_vpc_eip.test
  ]
}
```

**Parameter Description**:
- **name**: Assigned by referencing the instance_name in the input variable vpcs, used to specify the ECS instance name
- **availability_zone**: Assigned by referencing the first availability zone of the data source data.huaweicloud_availability_zones.test
- **flavor_id**: Prefers the ecs_flavor_id in the input variable vpcs; if not specified, uses the first flavor ID returned by the compute flavors data source
- **image_id**: Prefers the ecs_image_id in the input variable vpcs; if not specified, uses the image ID returned by the image data source
- **security_group_ids**: Assigned by referencing the ID of the security group resource huaweicloud_networking_secgroup.test
- **system_disk_type**: Assigned by referencing the ecs_system_disk_type in the input variable vpcs; defaults to SSD if not specified
- **system_disk_size**: Assigned by referencing the ecs_system_disk_size in the input variable vpcs; defaults to 40 GB if not specified
- **network.uuid**: Assigned by referencing the ID of the subnet resource huaweicloud_vpc_subnet.test
- **admin_pass**: Prefers the ecs_admin_password in the input variable vpcs; if not specified, uses the global input variable ecs_admin_password
- **tags**: Assigned by referencing the ecs_instance_tags in the input variable vpcs

### 6. Create VPC Peering Connection and Routes

Add the following script in the TF file (such as main.tf):

```hcl
# Create a VPC peering connection and routes in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "peering_connection_name" {
  description = "The name of the VPC peering connection"
  type        = string
  default     = "peering-connection-example"
}

variable "route_type" {
  description = "The type of the route"
  type        = string
  default     = "peering"
}

resource "huaweicloud_vpc_peering_connection" "cross_vpc_peering" {
  name        = var.peering_connection_name
  vpc_id      = huaweicloud_vpc.test[0].id
  peer_vpc_id = huaweicloud_vpc.test[1].id
}

resource "huaweicloud_vpc_route" "route_to_peer_vpc" {
  vpc_id      = huaweicloud_vpc.test[0].id
  destination = try(var.vpcs[1].vpc_cidr, null)
  type        = var.route_type
  nexthop     = huaweicloud_vpc_peering_connection.cross_vpc_peering.id
}

resource "huaweicloud_vpc_route" "route_other_to_internet_via_peering" {
  vpc_id      = huaweicloud_vpc.test[1].id
  destination = "0.0.0.0/0"
  type        = var.route_type
  nexthop     = huaweicloud_vpc_peering_connection.cross_vpc_peering.id
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable peering_connection_name, used to specify the peering connection name
- **vpc_id**: Assigned by referencing the ID of the VPC resource huaweicloud_vpc.test[0], representing the local VPC
- **peer_vpc_id**: Assigned by referencing the ID of the VPC resource huaweicloud_vpc.test[1], representing the peer VPC
- **destination**: In VPC One, assigned by referencing the vpcs[1].vpc_cidr in the input variable vpcs; in VPC Other, fixed to 0.0.0.0/0
- **type**: Assigned by referencing the input variable route_type, used to specify the route type
- **nexthop**: Assigned by referencing the ID of the peering connection resource huaweicloud_vpc_peering_connection.cross_vpc_peering

### 7. Create EIPs

Add the following script in the TF file (such as main.tf):

```hcl
# Create EIPs in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "eips" {
  description = "List of EIP configurations"
  type        = list(object({
    name_suffix = string
  }))
  default     = [
    {
      name_suffix = "snat"
    },
    {
      name_suffix = "dnat"
    }
  ]
}

variable "nat_gateway_name" {
  description = "The name of the NAT gateway"
  type        = string
  default     = "nat-gateway-example"
}

variable "bandwidth_size" {
  description = "The bandwidth size in Mbps"
  type        = number
  default     = 5
}

variable "bandwidth_charge_mode" {
  description = "The charge mode of the bandwidth"
  type        = string
  default     = "traffic"
}

resource "huaweicloud_vpc_eip" "test" {
  count = length(var.eips)

  publicip {
    type = "5_bgp"
  }

  bandwidth {
    share_type  = "PER"
    name        = "${var.nat_gateway_name}-${var.eips[count.index].name_suffix}-shared-bandwidth"
    size        = var.bandwidth_size
    charge_mode = var.bandwidth_charge_mode
  }
}
```

**Parameter Description**:
- **publicip.type**: Fixed to 5_bgp, indicating the use of a full dynamic BGP public IP
- **bandwidth.share_type**: Fixed to PER, indicating the use of dedicated bandwidth
- **bandwidth.name**: Assigned by combining the input variable nat_gateway_name and the name_suffix in eips, used to identify the bandwidth name
- **bandwidth.size**: Assigned by referencing the input variable bandwidth_size, used to specify the bandwidth size
- **bandwidth.charge_mode**: Assigned by referencing the input variable bandwidth_charge_mode, used to specify the bandwidth charge mode

### 8. Create NAT Gateway

Add the following script in the TF file (such as main.tf):

```hcl
# Create a NAT gateway in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "nat_gateway_spec" {
  description = "The specification of the NAT gateway"
  type        = string
  default     = "1"
}

resource "huaweicloud_nat_gateway" "test" {
  name        = var.nat_gateway_name
  description = "NAT gateway for VPC One to provide internet access"
  spec        = var.nat_gateway_spec
  vpc_id      = huaweicloud_vpc.test[0].id
  subnet_id   = huaweicloud_vpc_subnet.test[0].id
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable nat_gateway_name, used to specify the NAT gateway name
- **description**: Used to describe the purpose of the NAT gateway
- **spec**: Assigned by referencing the input variable nat_gateway_spec, used to specify the NAT gateway specification
- **vpc_id**: Assigned by referencing the ID of the VPC resource huaweicloud_vpc.test[0], indicating the NAT gateway is deployed in VPC One
- **subnet_id**: Assigned by referencing the ID of the subnet resource huaweicloud_vpc_subnet.test[0]

### 9. Create SNAT Rules

Add the following script in the TF file (such as main.tf):

```hcl
# Create SNAT rules in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "snat_rules" {
  description = "List of SNAT rule configurations"
  type        = list(object({
    eip_index    = number
    subnet_index = optional(number)
    source_type  = optional(number)
    vpc_index    = optional(number)
  }))
  default     = [
    {
      eip_index    = 0
      subnet_index = 0
      source_type  = null
      vpc_index    = null
    },
    {
      eip_index    = 0
      subnet_index = null
      source_type  = 1
      vpc_index    = 1
    }
  ]
}

resource "huaweicloud_nat_snat_rule" "test" {
  count = length(var.snat_rules)

  nat_gateway_id = huaweicloud_nat_gateway.test.id
  floating_ip_id = huaweicloud_vpc_eip.test[var.snat_rules[count.index].eip_index].id
  subnet_id      = var.snat_rules[count.index].subnet_index != null ? huaweicloud_vpc_subnet.test[var.snat_rules[count.index].subnet_index].id : null
  source_type    = var.snat_rules[count.index].source_type
  cidr           = var.snat_rules[count.index].vpc_index != null ? var.vpcs[var.snat_rules[count.index].vpc_index].subnet_cidr : null
}
```

**Parameter Description**:
- **nat_gateway_id**: Assigned by referencing the ID of the NAT gateway resource huaweicloud_nat_gateway.test
- **floating_ip_id**: Assigned by referencing the EIP ID specified by eip_index in the EIP resource huaweicloud_vpc_eip.test
- **subnet_id**: When subnet_index is specified in snat_rules, assigned by referencing the ID of the corresponding subnet resource huaweicloud_vpc_subnet.test
- **source_type**: Assigned by referencing the source_type in the input variable snat_rules, used to specify the SNAT rule source type
- **cidr**: When vpc_index is specified in snat_rules, assigned by referencing the corresponding subnet CIDR block in the input variable vpcs

### 10. Create DNAT Rules

Add the following script in the TF file (such as main.tf):

```hcl
# Create DNAT rules in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dnat_rules" {
  description = "List of DNAT rule configurations"
  type        = list(object({
    eip_index             = number
    instance_index        = number
    external_service_port = string
  }))
  default     = [
    {
      eip_index             = 1
      instance_index        = 0
      external_service_port = "8022"
    },
    {
      eip_index             = 1
      instance_index        = 1
      external_service_port = "8023"
    }
  ]
}

variable "dnat_protocol" {
  description = "The protocol for DNAT rules"
  type        = string
  default     = "tcp"
}

variable "dnat_internal_port" {
  description = "The internal service port for DNAT rules"
  type        = string
  default     = "22"
}

resource "huaweicloud_nat_dnat_rule" "test" {
  count = length(var.dnat_rules)

  nat_gateway_id        = huaweicloud_nat_gateway.test.id
  floating_ip_id        = huaweicloud_vpc_eip.test[var.dnat_rules[count.index].eip_index].id
  private_ip            = try(huaweicloud_compute_instance.test[var.dnat_rules[count.index].instance_index].network[0].fixed_ip_v4, null)
  protocol              = var.dnat_protocol
  internal_service_port = var.dnat_internal_port
  external_service_port = var.dnat_rules[count.index].external_service_port
}
```

**Parameter Description**:
- **nat_gateway_id**: Assigned by referencing the ID of the NAT gateway resource huaweicloud_nat_gateway.test
- **floating_ip_id**: Assigned by referencing the EIP ID specified by eip_index in the EIP resource huaweicloud_vpc_eip.test
- **private_ip**: Assigned by referencing the private IP of the ECS instance specified by instance_index in the ECS instance resource huaweicloud_compute_instance.test
- **protocol**: Assigned by referencing the input variable dnat_protocol, used to specify the DNAT rule protocol
- **internal_service_port**: Assigned by referencing the input variable dnat_internal_port, used to specify the internal service port
- **external_service_port**: Assigned by referencing the external_service_port in the input variable dnat_rules, used to specify the external service port

### 11. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication and region information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# ECS instance administrator password
ecs_admin_password = "YourSecurePassword123!"

# VPC configurations
vpcs = [
  {
    vpc_name          = "my-vpc-one"
    vpc_cidr          = "192.168.0.0/16"
    subnet_name       = "my-subnet-one"
    subnet_cidr       = "192.168.1.0/24"
    subnet_gateway_ip = "192.168.1.1"
    instance_name     = "my-instance-one"
  },
  {
    vpc_name          = "my-vpc-other"
    vpc_cidr          = "10.0.0.0/16"
    subnet_name       = "my-subnet-other"
    subnet_cidr       = "10.0.2.0/24"
    subnet_gateway_ip = "10.0.2.1"
    instance_name     = "my-instance-other"
  }
]

# NAT gateway configurations
nat_gateway_name = "my-nat-gateway"
bandwidth_size   = 10
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="nat_gateway_name=my-nat-gateway"`
2. Environment variables: `export TF_VAR_nat_gateway_name=my-nat-gateway`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the NAT gateway and VPC peering connection
4. Run `terraform show` to view the created NAT gateway and VPC peering connection

## Reference Information

- [Huawei Cloud NAT Gateway Product Documentation](https://support.huaweicloud.com/natgateway/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For NAT Gateway Shared via VPC Peering](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/nat/nat-gateway-vpc-peering)
