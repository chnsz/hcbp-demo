# Deploy Cross-Region IPv4 Network

## Application Scenario

Elastic IP (EIP) is an independently applicable and bindable public IP address resource provided by Huawei Cloud, offering cloud resources the ability to access the public network and be accessed from it. When services need a unified public access entry across regions, you can use Global EIP (G-EIP) together with global internet bandwidth and global connection bandwidth to connect cloud resources in different regions to the same public network egress, achieving cross-region network interconnection.

This best practice will introduce how to use Terraform to automatically deploy a cross-region IPv4 network, including VPC and subnet creation, security group and security group rule configuration, ECS instance creation, VPC internet gateway creation, global EIP pool query, global internet bandwidth creation, global EIP creation, and associating the global EIP with the ECS instance, the VPC internet gateway, and the global connection bandwidth.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS Flavors (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [Images (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)
- [Global EIP Pools (data.huaweicloud_global_eip_pools)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/global_eip_pools)
- [IAM Projects (data.huaweicloud_identity_projects)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Elastic Cloud Server (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [VPC Internet Gateway (huaweicloud_vpc_internet_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_internet_gateway)
- [Global Internet Bandwidth (huaweicloud_global_internet_bandwidth)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/global_internet_bandwidth)
- [Global EIP (huaweicloud_global_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/global_eip)
- [Global EIP Associate (huaweicloud_global_eip_associate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/global_eip_associate)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
data.huaweicloud_compute_flavors.test
data.huaweicloud_images_images.test
data.huaweicloud_global_eip_pools.test
data.huaweicloud_identity_projects.test
    └── huaweicloud_vpc.test
        └── huaweicloud_vpc_subnet.test
            └── huaweicloud_networking_secgroup.test
                └── huaweicloud_networking_secgroup_rule.test
                    └── huaweicloud_compute_instance.test
                        └── huaweicloud_vpc_internet_gateway.test
                            └── huaweicloud_global_internet_bandwidth.test
                                └── huaweicloud_global_eip.test
                                    └── huaweicloud_global_eip_associate.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query the Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones in the current region:

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}
```

### 3. Query the ECS Flavors

Add the following script in the TF file (such as main.tf) to query the ECS flavor list:

```hcl
# Query the ECS flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_flavor_id" {
  description = "The ID of the ECS instance flavor"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "The CPU core count of the ECS instance flavor"
  type        = number
  default     = 2
}

variable "instance_flavor_memory_size" {
  description = "The memory size of the ECS instance flavor"
  type        = number
  default     = 4
}

data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**Parameter description**:
- **count**: The flavor query is executed only when the instance flavor ID is not specified
- **availability_zone**: Assigned by referencing the first availability zone of the availability zones data source
- **performance_type**: Assigned by referencing the input variable instance_flavor_performance_type
- **cpu_core_count**: Assigned by referencing the input variable instance_flavor_cpu_core_count
- **memory_size**: Assigned by referencing the input variable instance_flavor_memory_size

### 4. Query the Images

Add the following script in the TF file (such as main.tf) to query the ECS image list:

```hcl
# Query the images in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_image_id" {
  description = "The ID of the ECS instance image"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_image_visibility" {
  description = "The visibility of the ECS instance image"
  type        = string
  default     = "public"
}

variable "instance_image_os" {
  description = "The OS of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}
```

**Parameter description**:
- **count**: The image query is executed only when the image ID is not specified
- **flavor_id**: Assigned by referencing the first flavor ID of the flavors data source when the instance flavor ID is not specified
- **visibility**: Assigned by referencing the input variable instance_image_visibility
- **os**: Assigned by referencing the input variable instance_image_os

### 5. Create a Virtual Private Cloud

Add the following script in the TF file (such as main.tf) to create a virtual private cloud:

```hcl
# Create a virtual private cloud in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter description**:
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, and the default enterprise project is used when it is not specified

### 6. Create a Virtual Private Cloud Subnet

Add the following script in the TF file (such as main.tf) to create a virtual private cloud subnet:

```hcl
# Create a virtual private cloud subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 4, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 4, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter description**:
- **vpc_id**: Assigned by referencing the ID of the virtual private cloud
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr, and the subnet CIDR is automatically calculated based on the VPC CIDR when it is not specified
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip, and the gateway IP is automatically calculated based on the subnet CIDR when it is not specified

### 7. Create a Security Group

Add the following script in the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**Parameter description**:
- **name**: Assigned by referencing the input variable security_group_name

### 8. Create Security Group Rules

Add the following script in the TF file (such as main.tf) to create security group rules:

```hcl
# Create security group rules in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_rule_configurations" {
  description = "The list of security group rule configurations"

  type = list(object({
    direction        = optional(string, "ingress")
    ethertype        = optional(string, "IPv4")
    protocol         = optional(string, null)
    ports            = optional(string, null)
    remote_ip_prefix = optional(string, "0.0.0.0/0")
  }))

  nullable = false
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = length(var.security_group_rule_configurations)

  direction         = lookup(var.security_group_rule_configurations[count.index], "direction", "ingress")
  ethertype         = lookup(var.security_group_rule_configurations[count.index], "ethertype", "IPv4")
  protocol          = lookup(var.security_group_rule_configurations[count.index], "protocol", null)
  ports             = lookup(var.security_group_rule_configurations[count.index], "ports", null)
  remote_ip_prefix  = lookup(var.security_group_rule_configurations[count.index], "remote_ip_prefix", "0.0.0.0/0")
  security_group_id = huaweicloud_networking_secgroup.test.id
}
```

**Parameter description**:
- **count**: Assigned by referencing the length of the input variable security_group_rule_configurations
- **direction**: Assigned by referencing the direction in the input variable security_group_rule_configurations
- **ethertype**: Assigned by referencing the ethertype in the input variable security_group_rule_configurations
- **protocol**: Assigned by referencing the protocol in the input variable security_group_rule_configurations
- **ports**: Assigned by referencing the ports in the input variable security_group_rule_configurations
- **remote_ip_prefix**: Assigned by referencing the remote_ip_prefix in the input variable security_group_rule_configurations
- **security_group_id**: Assigned by referencing the ID of the security group

### 9. Create an Elastic Cloud Server

Add the following script in the TF file (such as main.tf) to create an elastic cloud server:

```hcl
# Create an elastic cloud server in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "instance_administrator_password" {
  description = "The administrator password of the ECS instance"
  type        = string
  sensitive   = true
}

resource "huaweicloud_compute_instance" "test" {
  name               = var.instance_name
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, "") : var.instance_flavor_id
  image_id           = var.instance_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, "") : var.instance_image_id
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  admin_pass         = var.instance_administrator_password

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  depends_on = [huaweicloud_networking_secgroup_rule.test]

  lifecycle {
    ignore_changes = [
      availability_zone,
      flavor_id,
      image_id,
      admin_pass,
    ]
  }
}
```

**Parameter description**:
- **name**: Assigned by referencing the input variable instance_name
- **availability_zone**: Assigned by referencing the first availability zone of the availability zones data source
- **flavor_id**: Assigned by referencing the first flavor ID of the flavors data source when the instance flavor ID is not specified
- **image_id**: Assigned by referencing the first image ID of the images data source when the image ID is not specified
- **security_group_ids**: Assigned by referencing the ID of the security group
- **admin_pass**: Assigned by referencing the input variable instance_administrator_password
- **network.uuid**: Assigned by referencing the ID of the subnet

### 10. Create a VPC Internet Gateway

Add the following script in the TF file (such as main.tf) to create a VPC internet gateway:

```hcl
# Create a VPC internet gateway in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "internet_gateway_name" {
  description = "The name of the VPC internet gateway"
  type        = string
}

variable "internet_gateway_add_route" {
  description = "Whether to add route to the internet gateway"
  type        = bool
  default     = true
}

resource "huaweicloud_vpc_internet_gateway" "test" {
  vpc_id    = huaweicloud_vpc.test.id
  subnet_id = huaweicloud_vpc_subnet.test.id
  name      = var.internet_gateway_name
  add_route = var.internet_gateway_add_route
}
```

**Parameter description**:
- **vpc_id**: Assigned by referencing the ID of the virtual private cloud
- **subnet_id**: Assigned by referencing the ID of the subnet
- **name**: Assigned by referencing the input variable internet_gateway_name
- **add_route**: Assigned by referencing the input variable internet_gateway_add_route

### 11. Query the Global EIP Pools

Add the following script in the TF file (such as main.tf) to query the global EIP pool list:

```hcl
# Query the global EIP pools in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "global_eip_access_site" {
  description = "The access site used to filter the global EIP pool"
  type        = string
  default     = "cn-north-beijing"
  nullable    = false
}

variable "global_eip_ip_version" {
  description = "The IP version of the global EIP"
  type        = string
  default     = "4"
}

data "huaweicloud_global_eip_pools" "test" {
  access_site = var.global_eip_access_site != "" ? var.global_eip_access_site : null
  ip_version  = var.global_eip_ip_version
}
```

**Parameter description**:
- **access_site**: Assigned by referencing the input variable global_eip_access_site
- **ip_version**: Assigned by referencing the input variable global_eip_ip_version

### 12. Create a Global Internet Bandwidth

Add the following script in the TF file (such as main.tf) to create a global internet bandwidth:

```hcl
# Create a global internet bandwidth in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "internet_bandwidth_charge_mode" {
  description = "The charge mode of the global internet bandwidth"
  type        = string
  default     = "95peak_guar"
}

variable "internet_bandwidth_size" {
  description = "The size of the global internet bandwidth in Mbit/s"
  type        = number
  default     = 300
}

variable "internet_bandwidth_name" {
  description = "The name of the global internet bandwidth"
  type        = string
  default     = null
}

variable "internet_bandwidth_ingress_size" {
  description = "The ingress size of the global internet bandwidth in Mbit/s"
  type        = number
  default     = null
}

variable "internet_bandwidth_tags" {
  description = "The tags of the internet bandwidth"
  type        = map(string)
  default     = null
}

resource "huaweicloud_global_internet_bandwidth" "test" {
  access_site           = try(data.huaweicloud_global_eip_pools.test.geip_pools[0].access_site, null)
  charge_mode           = var.internet_bandwidth_charge_mode
  size                  = var.internet_bandwidth_size
  isp                   = try(data.huaweicloud_global_eip_pools.test.geip_pools[0].isp, null)
  name                  = var.internet_bandwidth_name
  ingress_size          = var.internet_bandwidth_ingress_size
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  tags                  = var.internet_bandwidth_tags
}
```

**Parameter description**:
- **access_site**: Assigned by referencing the access site of the global EIP pools data source
- **charge_mode**: Assigned by referencing the input variable internet_bandwidth_charge_mode
- **size**: Assigned by referencing the input variable internet_bandwidth_size
- **isp**: Assigned by referencing the ISP information of the global EIP pools data source
- **name**: Assigned by referencing the input variable internet_bandwidth_name
- **ingress_size**: Assigned by referencing the input variable internet_bandwidth_ingress_size
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, and the default enterprise project is used when it is not specified
- **tags**: Assigned by referencing the input variable internet_bandwidth_tags

### 13. Create a Global EIP

Add the following script in the TF file (such as main.tf) to create a global EIP:

```hcl
# Create a global EIP in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "global_eip_name" {
  description = "The name of the global EIP"
  type        = string
}

variable "global_eip_description" {
  description = "The description of the global EIP"
  type        = string
  default     = ""
}

variable "global_eip_tags" {
  description = "The tags of the global EIP"
  type        = map(string)
  default     = null
}

resource "huaweicloud_global_eip" "test" {
  access_site           = huaweicloud_global_internet_bandwidth.test.access_site
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  geip_pool_name        = try(data.huaweicloud_global_eip_pools.test.geip_pools[0].name, null)
  internet_bandwidth_id = huaweicloud_global_internet_bandwidth.test.id
  name                  = var.global_eip_name
  description           = var.global_eip_description
  tags                  = var.global_eip_tags
}
```

**Parameter description**:
- **access_site**: Assigned by referencing the access site of the global internet bandwidth
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, and the default enterprise project is used when it is not specified
- **geip_pool_name**: Assigned by referencing the pool name of the global EIP pools data source
- **internet_bandwidth_id**: Assigned by referencing the ID of the global internet bandwidth
- **name**: Assigned by referencing the input variable global_eip_name
- **description**: Assigned by referencing the input variable global_eip_description
- **tags**: Assigned by referencing the input variable global_eip_tags

### 14. Query the IAM Projects

Add the following script in the TF file (such as main.tf) to query the IAM projects of the region where the ECS instance is located:

```hcl
# Query the IAM projects in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_identity_projects" "test" {
  name = huaweicloud_compute_instance.test.region
}
```

**Parameter description**:
- **name**: Assigned by referencing the region where the elastic cloud server is located

### 15. Create a Global EIP Associate

Add the following script in the TF file (such as main.tf) to associate the global EIP with the ECS instance, the VPC internet gateway, and the global connection bandwidth:

```hcl
# Create a global EIP associate in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "gc_bandwidth_name" {
  description = "The name of the global connection bandwidth"
  type        = string
}

variable "gc_bandwidth_charge_mode" {
  description = "The charge mode of the global connection bandwidth"
  type        = string
  default     = "95"
}

variable "gc_bandwidth_size" {
  description = "The size of the global connection bandwidth in Mbit/s"
  type        = number
  default     = 100
}

resource "huaweicloud_global_eip_associate" "test" {
  global_eip_id  = huaweicloud_global_eip.test.id
  is_reserve_gcb = false

  associate_instance {
    region        = huaweicloud_compute_instance.test.region
    project_id    = try(data.huaweicloud_identity_projects.test.projects[0].id, null)
    instance_type = "ECS"
    instance_id   = huaweicloud_compute_instance.test.id
  }

  gc_bandwidth {
    name                  = var.gc_bandwidth_name
    charge_mode           = var.gc_bandwidth_charge_mode
    size                  = var.gc_bandwidth_size
    enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  }

  depends_on = [huaweicloud_vpc_internet_gateway.test]
}
```

**Parameter description**:
- **global_eip_id**: Assigned by referencing the ID of the global EIP
- **is_reserve_gcb**: Whether to reserve the global connection bandwidth, which is set to false here
- **associate_instance.region**: Assigned by referencing the region where the elastic cloud server is located
- **associate_instance.project_id**: Assigned by referencing the first project ID of the IAM projects data source
- **associate_instance.instance_type**: The type of the associated instance, which is set to ECS here
- **associate_instance.instance_id**: Assigned by referencing the ID of the elastic cloud server
- **gc_bandwidth.name**: Assigned by referencing the input variable gc_bandwidth_name
- **gc_bandwidth.charge_mode**: Assigned by referencing the input variable gc_bandwidth_charge_mode
- **gc_bandwidth.size**: Assigned by referencing the input variable gc_bandwidth_size
- **gc_bandwidth.enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, and the default enterprise project is used when it is not specified

### 16. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"

security_group_rule_configurations = [
  {
    direction        = "ingress"
    ethertype        = "IPv4"
    protocol         = "icmp"
    remote_ip_prefix = "0.0.0.0/0"
  },
  {
    direction        = "ingress"
    ethertype        = "IPv4"
    protocol         = "tcp"
    ports            = "22,3389"
    remote_ip_prefix = "10.1.0.7/32"
  },
  {
    direction        = "egress"
    ethertype        = "IPv4"
    remote_ip_prefix = "0.0.0.0/0"
  },
]

instance_name                   = "tf_test_ecs"
internet_gateway_name           = "tf_test_igw"
global_eip_name                 = "tf_test_geip"
internet_bandwidth_name         = "tf_test_internet_bandwidth"
gc_bandwidth_name               = "tf_test_gc_bandwidth"
instance_administrator_password = "YourPassword@123"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 17. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the cross-region IPv4 network
4. Run `terraform show` to view the created cross-region IPv4 network

## Reference Information

- [Huawei Cloud Elastic IP Product Documentation](https://support.huaweicloud.com/eip/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EIP Cross-Region IPv4 Network](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eip/geip-ipv4-cross-region)
