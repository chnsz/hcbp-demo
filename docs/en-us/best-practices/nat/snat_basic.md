# Deploy SNAT Rule

## Application Scenario

NAT Gateway is a network address translation service provided by Huawei Cloud. It enables cloud resources in a Virtual Private Cloud (VPC) to access the internet without requiring each server to have a public IP address. An SNAT (Source Network Address Translation) rule translates traffic from a subnet or a specified CIDR block in the VPC into the Elastic IP bound to the NAT gateway, so that internal instances can share a public egress to access the internet while internal network addresses remain hidden.

This best practice will introduce how to use Terraform to automatically deploy an SNAT rule, including VPC and subnet creation, NAT gateway creation, EIP creation, SNAT rule configuration, and the ECS instance and security group configuration used to verify outbound access.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Compute Flavors (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [Images (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [NAT Gateway (huaweicloud_nat_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_gateway)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [SNAT Rule (huaweicloud_nat_snat_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_snat_rule)
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
            │       └── huaweicloud_nat_snat_rule
            └── huaweicloud_compute_instance

huaweicloud_vpc_eip
    └── huaweicloud_nat_snat_rule

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
    └── huaweicloud_compute_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a VPC and a Subnet

Add the following script in the TF file (such as main.tf):

```hcl
# Create VPC and subnet resources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing `huaweicloud_vpc.test.id`
- **name** (subnet): The subnet name, assigned by referencing the input variable subnet_name
- **cidr** (subnet): The subnet CIDR block, assigned by referencing the input variable subnet_cidr
- **gateway_ip**: The subnet gateway IP; when the input variable subnet_gateway_ip is empty, the gateway address is calculated automatically based on the VPC CIDR block

### 3. Create a NAT Gateway

Add the following script in the TF file (such as main.tf):

```hcl
# Create a NAT gateway resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "nat_gateway_name" {
  description = "The name of the NAT gateway"
  type        = string
}

variable "gateway_description" {
  description = "The description of the NAT gateway"
  type        = string
  default     = ""
}

variable "gateway_spec" {
  description = "The specification of the NAT gateway"
  type        = string
  default     = "1"

  validation {
    condition     = contains(["1", "2", "3", "4"], var.gateway_spec)
    error_message = "The gateway_spec must be one of: 1, 2, 3, 4."
  }
}

resource "huaweicloud_nat_gateway" "test" {
  name        = var.nat_gateway_name
  description = var.gateway_description
  spec        = var.gateway_spec
  vpc_id      = huaweicloud_vpc.test.id
  subnet_id   = huaweicloud_vpc_subnet.test.id
}
```

**Parameter Description**:
- **name**: The NAT gateway name, assigned by referencing the input variable nat_gateway_name
- **description**: The NAT gateway description, assigned by referencing the input variable gateway_description
- **spec**: The NAT gateway specification, assigned by referencing the input variable gateway_spec; valid values are 1 (small), 2 (medium), 3 (large), and 4 (extra-large)
- **vpc_id**: The ID of the VPC to which the NAT gateway belongs, referencing `huaweicloud_vpc.test.id`
- **subnet_id**: The ID of the subnet to which the NAT gateway belongs, referencing `huaweicloud_vpc_subnet.test.id`

### 4. Create an Elastic IP

Add the following script in the TF file (such as main.tf):

```hcl
# Create an EIP resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "eip_bandwidth_name" {
  description = "The name of the EIP bandwidth"
  type        = string
}

variable "eip_bandwidth_size" {
  description = "The size of the EIP bandwidth in Mbps"
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
- **type**: The EIP type, fixed to `5_bgp`
- **name**: The bandwidth name, assigned by referencing the input variable eip_bandwidth_name
- **size**: The bandwidth size in Mbps, assigned by referencing the input variable eip_bandwidth_size
- **share_type**: The bandwidth share type, assigned by referencing the input variable eip_bandwidth_share_type; valid values are PER (dedicated) and WHOLE (shared)
- **charge_mode**: The bandwidth charge mode, assigned by referencing the input variable eip_bandwidth_charge_mode; valid values are traffic and bandwidth

### 5. Create an SNAT Rule

Add the following script in the TF file (such as main.tf):

```hcl
# Create an SNAT rule resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "snat_source_type" {
  description = "The resource type of the SNAT rule"
  type        = number
  default     = 0
  nullable    = false

  validation {
    condition     = contains([0, 1], var.snat_source_type)
    error_message = "The snat_source_type must be one of: 0 (VPC scenario), 1 (Direct Connect scenario)."
  }
}

variable "snat_cidr" {
  description = "The CIDR block connected by SNAT rule (DC side, required when snat_source_type is 1)"
  type        = string
  default     = ""
  nullable    = false
}

variable "snat_description" {
  description = "The description of the SNAT rule"
  type        = string
  default     = ""
}

resource "huaweicloud_nat_snat_rule" "test" {
  nat_gateway_id = huaweicloud_nat_gateway.test.id
  floating_ip_id = huaweicloud_vpc_eip.test.id
  source_type    = var.snat_source_type
  subnet_id      = var.snat_source_type == 0 ? huaweicloud_vpc_subnet.test.id : null
  cidr           = var.snat_source_type == 1 ? var.snat_cidr : null
  description    = var.snat_description
}
```

**Parameter Description**:
- **nat_gateway_id**: The ID of the NAT gateway to which the SNAT rule belongs, referencing `huaweicloud_nat_gateway.test.id`
- **floating_ip_id**: The ID of the EIP used by the SNAT rule, referencing `huaweicloud_vpc_eip.test.id`
- **source_type**: The source type of the SNAT rule, assigned by referencing the input variable snat_source_type; 0 indicates the VPC scenario and 1 indicates the Direct Connect scenario
- **subnet_id**: When source_type is 0, the subnet is specified by referencing `huaweicloud_vpc_subnet.test.id`
- **cidr**: When source_type is 1, the Direct Connect side CIDR block is specified by referencing the input variable snat_cidr
- **description**: The SNAT rule description, assigned by referencing the input variable snat_description

### 6. Query Availability Zones, Compute Flavors, and Images

Add the following script in the TF file (such as main.tf):

```hcl
# Query availability zones, compute flavors, and images in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "ecs_flavor_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "ecs_flavor_cpu_core_count" {
  description = "The CPU core count of the ECS instance flavor"
  type        = number
  default     = 2
}

variable "ecs_flavor_memory_size" {
  description = "The memory size of the ECS instance flavor"
  type        = number
  default     = 4
}

variable "ecs_flavor_id" {
  description = "The ID of the ECS instance flavor"
  type        = string
  default     = ""
  nullable    = false
}

variable "ecs_image_visibility" {
  description = "The visibility of the ECS instance image"
  type        = string
  default     = "public"
}

variable "ecs_image_os" {
  description = "The OS of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

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
- **availability_zone**: The availability zone used for querying compute flavors, taking the first availability zone from the availability zones data source
- **performance_type**: The performance type of the compute flavor, assigned by referencing the input variable ecs_flavor_performance_type
- **cpu_core_count**: The CPU core count of the compute flavor, assigned by referencing the input variable ecs_flavor_cpu_core_count
- **memory_size**: The memory size of the compute flavor in GB, assigned by referencing the input variable ecs_flavor_memory_size
- **flavor_id**: The compute flavor ID used for querying images; the input variable ecs_flavor_id takes precedence, and when it is empty the first flavor from the compute flavors data source is used
- **visibility**: The image visibility, assigned by referencing the input variable ecs_image_visibility
- **os**: The image OS, assigned by referencing the input variable ecs_image_os

### 7. Create a Security Group and a Rule

Add the following script in the TF file (such as main.tf):

```hcl
# Create a security group and a rule in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "ecs_security_group_name" {
  description = "The name of the security group for ECS instance"
  type        = string
  default     = "terraform-test-ecs-sg"
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.ecs_security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "egress"
  ethertype         = "IPv4"
  remote_ip_prefix  = "0.0.0.0/0"
  priority          = 1
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable ecs_security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group, set to `true`
- **security_group_id**: The ID of the security group to which the rule belongs, referencing `huaweicloud_networking_secgroup.test.id`
- **direction**: The rule direction, set to `egress` for outbound traffic
- **ethertype**: The network type, set to `IPv4`
- **remote_ip_prefix**: The remote IP CIDR block, set to `0.0.0.0/0` to allow access to any public address
- **priority**: The rule priority, set to `1`

### 8. Create an ECS Instance

Add the following script in the TF file (such as main.tf):

```hcl
# Create an ECS instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "ecs_instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "ecs_image_id" {
  description = "The ID of the ECS instance image"
  type        = string
  default     = ""
  nullable    = false
}

variable "ecs_system_disk_type" {
  description = "The type of the ECS instance system disk"
  type        = string
  default     = "SSD"
}

variable "ecs_system_disk_size" {
  description = "The size of the ECS instance system disk in GB"
  type        = number
  default     = 40
}

variable "ecs_admin_password" {
  description = "The password of the ECS instance administrator"
  type        = string
  sensitive   = true
}

variable "ecs_instance_tags" {
  description = "The tags of the ECS instance"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)
  flavor_id          = var.ecs_flavor_id != "" ? var.ecs_flavor_id : try(data.huaweicloud_compute_flavors.test.flavors[0].id, null)
  image_id           = var.ecs_image_id != "" ? var.ecs_image_id : try(data.huaweicloud_images_images.test.images[0].id, null)
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
- **name**: The ECS instance name, assigned by referencing the input variable ecs_instance_name
- **availability_zone**: The availability zone of the ECS instance, taking the first availability zone from the availability zones data source
- **flavor_id**: The ECS instance flavor ID; the input variable ecs_flavor_id takes precedence, and when it is empty the first flavor from the compute flavors data source is used
- **image_id**: The ECS instance image ID; the input variable ecs_image_id takes precedence, and when it is empty the first image from the images data source is used
- **security_group_ids**: The list of security group IDs bound to the ECS instance, referencing `huaweicloud_networking_secgroup.test.id`
- **system_disk_type**: The system disk type, assigned by referencing the input variable ecs_system_disk_type
- **system_disk_size**: The system disk size in GB, assigned by referencing the input variable ecs_system_disk_size
- **uuid**: The ID of the subnet to which the ECS instance belongs, referencing `huaweicloud_vpc_subnet.test.id`
- **admin_pass**: The administrator password of the ECS instance, assigned by referencing the input variable ecs_admin_password
- **tags**: The tags of the ECS instance, assigned by referencing the input variable ecs_instance_tags

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# VPC configuration
vpc_name = "terraform-test-vpc"
vpc_cidr = "192.168.0.0/16"

# Subnet configuration
subnet_name       = "terraform-test-subnet"
subnet_cidr       = "192.168.1.0/24"
subnet_gateway_ip = "192.168.1.1"

# NAT gateway configuration
nat_gateway_name    = "terraform-test-nat-gateway"
gateway_spec        = "1"
gateway_description = "NAT gateway for terraform test"

# EIP configuration
eip_bandwidth_name        = "terraform-test-eip-bandwidth"
eip_bandwidth_size        = 5
eip_bandwidth_share_type  = "PER"
eip_bandwidth_charge_mode = "traffic"

# SNAT rule configuration (VPC scenario)
snat_source_type = 0
snat_cidr        = "192.168.0.0/16"
snat_description = "SNAT rule for terraform test"

# ECS instance configuration
ecs_instance_name           = "terraform-test-ecs"
ecs_flavor_id               = ""
ecs_flavor_performance_type = "normal"
ecs_flavor_cpu_core_count   = 2
ecs_flavor_memory_size      = 4
ecs_image_id                = ""
ecs_image_visibility        = "public"
ecs_image_os                = "Ubuntu"
ecs_security_group_name     = "terraform-test-ecs-sg"
ecs_admin_password          = "YourPassword123!"
ecs_system_disk_type        = "GPSSD"
ecs_system_disk_size        = 40
ecs_instance_tags = {
  Environment = "test"
  Project     = "terraform-example"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before the tfvars name, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the SNAT rule
4. Run `terraform show` to view the created SNAT rule

## Reference Information

- [Huawei Cloud NAT Gateway Product Documentation](https://support.huaweicloud.com/natgateway/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For NAT Gateway SNAT Rule](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/nat/snat-basic)
