# Deploy DNAT Rule

## Application Scenario

NAT Gateway is a network address translation service provided by Huawei Cloud, supporting both SNAT and DNAT. DNAT (Destination Network Address Translation) maps a specified port on a public EIP to a port on a backend server within a VPC, so that clients on the internet can access services in the VPC through the EIP without binding a public IP directly to the backend server.

This best practice will introduce how to use Terraform to automatically deploy a DNAT rule, including VPC and subnet creation, NAT gateway creation, EIP creation, backend ECS instance and security group configuration, and DNAT rule creation with port mapping configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS Flavors (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [Images (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [NAT Gateway (huaweicloud_nat_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_gateway)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [DNAT Rule (huaweicloud_nat_dnat_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_dnat_rule)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [ECS Instance (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_compute_flavors
    │       └── data.huaweicloud_images_images
    │               └── huaweicloud_compute_instance
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            ├── huaweicloud_nat_gateway
            │       └── huaweicloud_nat_dnat_rule
            └── huaweicloud_compute_instance
                    └── huaweicloud_nat_dnat_rule

huaweicloud_vpc_eip
    └── huaweicloud_nat_dnat_rule

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_compute_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a VPC and Subnet

Add the following script in the TF file (such as main.tf):

```hcl
# Create VPC and subnet resources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The VPC name"
  type        = string
  default     = "vpc-dnat-basic"
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "172.16.0.0/16"
}

variable "subnet_name" {
  description = "The subnet name"
  type        = string
  default     = "subnet-dnat-basic"
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "172.16.10.0/24"
}

variable "subnet_gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = true
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing `huaweicloud_vpc.test.id`
- **cidr**: The CIDR block of the subnet, assigned by referencing the input variable subnet_cidr; calculated from the VPC CIDR if not specified
- **gateway_ip**: The gateway IP of the subnet, assigned by referencing the input variable subnet_gateway_ip; calculated from the subnet CIDR if not specified

### 3. Create a NAT Gateway

Add the following script in the TF file (such as main.tf):

```hcl
# Create a NAT gateway resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "nat_gateway_name" {
  description = "The NAT gateway name"
  type        = string
  default     = "nat-gateway-dnat-basic"
}

variable "nat_gateway_description" {
  description = "The description of the NAT gateway"
  type        = string
  default     = ""
}

variable "nat_gateway_spec" {
  description = "The specification of the NAT gateway"
  type        = string
  default     = "1"

  validation {
    condition     = contains(["1", "2", "3", "4"], var.nat_gateway_spec)
    error_message = "The nat_gateway_spec must be one of: 1, 2, 3, 4."
  }
}

resource "huaweicloud_nat_gateway" "test" {
  name        = var.nat_gateway_name
  description = var.nat_gateway_description
  spec        = var.nat_gateway_spec
  vpc_id      = huaweicloud_vpc.test.id
  subnet_id   = huaweicloud_vpc_subnet.test.id
}
```

**Parameter Description**:
- **name**: The NAT gateway name, assigned by referencing the input variable nat_gateway_name
- **description**: The NAT gateway description, assigned by referencing the input variable nat_gateway_description
- **spec**: The NAT gateway specification, assigned by referencing the input variable nat_gateway_spec; valid values are 1 (small), 2 (medium), 3 (large), and 4 (extra-large)
- **vpc_id**: The ID of the VPC to which the NAT gateway belongs, referencing `huaweicloud_vpc.test.id`
- **subnet_id**: The ID of the subnet to which the NAT gateway belongs, referencing `huaweicloud_vpc_subnet.test.id`

### 4. Create an Elastic IP

Add the following script in the TF file (such as main.tf):

```hcl
# Create an elastic IP resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "eip_bandwidth_name" {
  description = "The name of the EIP bandwidth"
  type        = string
  default     = "eip-dnat-basic"
}

variable "eip_bandwidth_size" {
  description = "The size of the EIP bandwidth (Mbit/s)"
  type        = number
  default     = 5
}

variable "eip_bandwidth_share_type" {
  description = "The share type of the EIP bandwidth"
  type        = string
  default     = "PER"

  validation {
    condition     = contains(["PER", "WHOLE"], var.eip_bandwidth_share_type)
    error_message = "The eip_bandwidth_share_type must be one of: PER, WHOLE."
  }
}

variable "eip_bandwidth_charge_mode" {
  description = "The charge mode of the EIP bandwidth"
  type        = string
  default     = "traffic"

  validation {
    condition     = contains(["traffic", "bandwidth"], var.eip_bandwidth_charge_mode)
    error_message = "The eip_bandwidth_charge_mode must be one of: traffic, bandwidth."
  }
}

resource "huaweicloud_vpc_eip" "test" {
  publicip {
    type = "5_bgp"
  }

  bandwidth {
    name        = var.eip_bandwidth_name
    size        = var.eip_bandwidth_size
    share_type  = var.eip_bandwidth_share_type
    charge_mode = var.eip_bandwidth_charge_mode
  }
}
```

**Parameter Description**:
- **publicip.type**: The EIP type, fixed to `5_bgp`
- **bandwidth.name**: The bandwidth name, assigned by referencing the input variable eip_bandwidth_name
- **bandwidth.size**: The bandwidth size (Mbit/s), assigned by referencing the input variable eip_bandwidth_size
- **bandwidth.share_type**: The bandwidth share type, assigned by referencing the input variable eip_bandwidth_share_type
- **bandwidth.charge_mode**: The bandwidth charge mode, assigned by referencing the input variable eip_bandwidth_charge_mode

### 5. Query Availability Zones, ECS Flavors, and Images

Add the following script in the TF file (such as main.tf):

```hcl
# Query availability zones, ECS flavors, and images in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_compute_flavors" "test" {
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.ecs_flavor_performance_type
  cpu_core_count    = var.ecs_flavor_cpu_core_count
  memory_size       = var.ecs_flavor_memory_size
}

data "huaweicloud_images_images" "test" {
  flavor_id  = var.ecs_flavor_id != "" ? var.ecs_flavor_id : try(data.huaweicloud_compute_flavors.test.flavors[0].id, null)
  visibility = var.ecs_image_visibility
  os         = var.ecs_image_os
}
```

**Parameter Description**:
- **availability_zone**: The availability zone name, using the first availability zone in the list
- **performance_type**: The performance type of the ECS flavor, assigned by referencing the input variable ecs_flavor_performance_type
- **cpu_core_count**: The CPU core count of the ECS flavor, assigned by referencing the input variable ecs_flavor_cpu_core_count
- **memory_size**: The memory size (GB) of the ECS flavor, assigned by referencing the input variable ecs_flavor_memory_size
- **flavor_id**: The flavor ID used for image query, preferring the input variable ecs_flavor_id, or the queried flavor if not specified
- **visibility**: The image visibility, assigned by referencing the input variable ecs_image_visibility
- **os**: The image OS, assigned by referencing the input variable ecs_image_os

### 6. Create a Security Group and Security Group Rules

Add the following script in the TF file (such as main.tf):

```hcl
# Create security group and security group rule resources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name of the backend instance"
  type        = string
  default     = "sg-dnat-backend"
}

variable "backend_protocol" {
  description = "The protocol used between the NAT gateway and backend ECS instance"
  type        = string
  default     = "tcp"
}

variable "backend_port" {
  description = "The port on the backend ECS instance that receives DNAT traffic"
  type        = number
  default     = 22
}

variable "ingress_cidr" {
  description = "The CIDR block that is allowed to access the DNAT service from the Internet"
  type        = string
  default     = "0.0.0.0/0"
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "ingress" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = var.backend_protocol
  port_range_min    = var.backend_port
  port_range_max    = var.backend_port
  remote_ip_prefix  = var.ingress_cidr
}

resource "huaweicloud_networking_secgroup_rule" "egress" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "egress"
  ethertype         = "IPv4"
  remote_ip_prefix  = "0.0.0.0/0"
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to `true` to keep only custom rules
- **direction**: The rule direction, `ingress` for inbound and `egress` for outbound
- **protocol**: The inbound rule protocol, assigned by referencing the input variable backend_protocol
- **port_range_min/port_range_max**: The inbound rule port range, assigned by referencing the input variable backend_port
- **remote_ip_prefix**: The inbound rule source CIDR block, assigned by referencing the input variable ingress_cidr

### 7. Create a Backend ECS Instance

Add the following script in the TF file (such as main.tf):

```hcl
# Create a backend ECS instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the backend ECS instance"
  type        = string
  default     = "ecs-dnat-backend"
}

variable "ecs_flavor_id" {
  description = "The flavor ID of the backend ECS instance"
  type        = string
  default     = ""
  nullable    = true
}

variable "ecs_image_id" {
  description = "The image ID of the backend ECS instance"
  type        = string
  default     = ""
  nullable    = true
}

variable "ecs_system_disk_type" {
  description = "The system disk type of the ECS instance"
  type        = string
  default     = "SSD"
}

variable "ecs_system_disk_size" {
  description = "The system disk size of the ECS instance (GB)"
  type        = number
  default     = 40
}

variable "ecs_admin_password" {
  description = "The administrator password of the ECS instance"
  type        = string
  sensitive   = true
  default     = ""
}

variable "ecs_instance_tags" {
  description = "The tags of the ECS instance"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_compute_instance" "test" {
  name               = var.instance_name
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)
  flavor_id          = var.ecs_flavor_id != "" ? var.ecs_flavor_id : try(data.huaweicloud_compute_flavors.test.flavors[0].id, "")
  image_id           = var.ecs_image_id != "" ? var.ecs_image_id : try(data.huaweicloud_images_images.test.images[0].id, "")
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  system_disk_type   = var.ecs_system_disk_type
  system_disk_size   = var.ecs_system_disk_size

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  admin_pass = var.ecs_admin_password
  tags       = var.ecs_instance_tags

  depends_on = [
    huaweicloud_nat_gateway.test,
    huaweicloud_vpc_eip.test
  ]
}
```

**Parameter Description**:
- **name**: The ECS instance name, assigned by referencing the input variable instance_name
- **availability_zone**: The availability zone of the ECS instance, using the first availability zone in the list
- **flavor_id**: The ECS instance flavor ID, preferring the input variable ecs_flavor_id, or the queried flavor if not specified
- **image_id**: The ECS instance image ID, preferring the input variable ecs_image_id, or the queried image if not specified
- **security_group_ids**: The list of security group IDs bound to the ECS instance, referencing `huaweicloud_networking_secgroup.test.id`
- **system_disk_type**: The system disk type, assigned by referencing the input variable ecs_system_disk_type
- **system_disk_size**: The system disk size (GB), assigned by referencing the input variable ecs_system_disk_size
- **network.uuid**: The ID of the subnet to which the ECS instance belongs, referencing `huaweicloud_vpc_subnet.test.id`
- **admin_pass**: The administrator password of the ECS instance, assigned by referencing the input variable ecs_admin_password
- **tags**: The tags of the ECS instance, assigned by referencing the input variable ecs_instance_tags

### 8. Create a DNAT Rule

Add the following script in the TF file (such as main.tf):

```hcl
# Create a DNAT rule resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "frontend_protocol" {
  description = "The protocol used between the client and NAT gateway"
  type        = string
  default     = "tcp"

  validation {
    condition     = contains(["tcp", "udp", "any"], var.frontend_protocol)
    error_message = "The frontend_protocol must be one of: tcp, udp, any."
  }
}

variable "frontend_port" {
  description = "The port on the public EIP that clients use to access the DNAT service"
  type        = number
  default     = 22
}

resource "huaweicloud_nat_dnat_rule" "test" {
  nat_gateway_id        = huaweicloud_nat_gateway.test.id
  floating_ip_id        = huaweicloud_vpc_eip.test.id
  port_id               = try(huaweicloud_compute_instance.test.network[0].port, null)
  protocol              = var.frontend_protocol
  internal_service_port = var.backend_port
  external_service_port = var.frontend_port
}
```

**Parameter Description**:
- **nat_gateway_id**: The ID of the NAT gateway to which the DNAT rule belongs, referencing `huaweicloud_nat_gateway.test.id`
- **floating_ip_id**: The ID of the EIP bound to the DNAT rule, referencing `huaweicloud_vpc_eip.test.id`
- **port_id**: The NIC port ID of the backend ECS instance, referencing `huaweicloud_compute_instance.test.network[0].port`
- **protocol**: The protocol between the client and the NAT gateway, assigned by referencing the input variable frontend_protocol
- **internal_service_port**: The port on the backend ECS instance that receives DNAT traffic, assigned by referencing the input variable backend_port
- **external_service_port**: The EIP port that clients use to access the DNAT service, assigned by referencing the input variable frontend_port

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# VPC and subnet
vpc_name          = "vpc-dnat-basic"
vpc_cidr          = "172.16.0.0/16"
subnet_name       = "subnet-dnat-basic"
subnet_cidr       = "172.16.0.0/24"
subnet_gateway_ip = "172.16.0.1"

# Backend ECS
security_group_name = "sg-dnat-backend"
ingress_cidr        = "192.168.0.0/16"
instance_name       = "ecs-dnat-backend"

# Optional: data source query parameters (used when IDs are not specified)
ecs_flavor_performance_type = "normal"
ecs_flavor_cpu_core_count   = 2
ecs_flavor_memory_size      = 4
ecs_image_visibility        = "public"
ecs_image_os                = "Ubuntu"
ecs_system_disk_type        = "SSD"
ecs_system_disk_size        = 40

# DNAT ports and protocol
backend_protocol  = "tcp"
backend_port      = 2288
frontend_protocol = "tcp"
frontend_port     = 2288

# EIP and NAT gateway
eip_bandwidth_name        = "eip-dnat-basic"
eip_bandwidth_size        = 5
eip_bandwidth_share_type  = "PER"
eip_bandwidth_charge_mode = "traffic"
nat_gateway_name          = "nat-gateway-dnat-basic"
nat_gateway_description   = ""
nat_gateway_spec          = "1"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DNAT rule
4. Run `terraform show` to view the created DNAT rule

## Reference Information

- [Huawei Cloud NAT Gateway Product Documentation](https://support.huaweicloud.com/natgateway/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For NAT Gateway DNAT Rule](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/nat/dnat-basic)
