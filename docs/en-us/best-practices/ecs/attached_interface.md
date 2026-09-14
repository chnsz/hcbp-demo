# Deploy Instance with Network Interface

## Application Scenario

Elastic Cloud Server (ECS) is an elastic computing service provided by Huawei Cloud. When services require multiple NICs, multi-subnet isolation, or independent security policies, you can attach additional network interfaces to an ECS instance. By attaching network interfaces, an instance can communicate in different subnets simultaneously, meeting scenarios such as high availability, traffic isolation, and flexible networking.

This best practice will introduce how to use Terraform to automatically deploy an ECS instance with an attached network interface, including querying availability zones, flavors, and images, creating a VPC and subnets, configuring a security group, creating the instance, and attaching the network interface.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS Flavors (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS Images (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Elastic Cloud Server (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [Compute Interface Attach (huaweicloud_compute_interface_attach)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_interface_attach)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_compute_instance

data.huaweicloud_compute_flavors
    └── huaweicloud_compute_instance

data.huaweicloud_images_images
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_compute_instance
        └── huaweicloud_compute_interface_attach

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance

huaweicloud_compute_instance
    └── huaweicloud_compute_interface_attach
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zone, ECS Flavor, and Image Information

Add the following script in the TF file (such as main.tf) to query the availability zone, flavor, and image information required for deploying the ECS instance:

```hcl
# Query availability zones, ECS flavors, and images in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone to which the ECS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_id" {
  description = "The flavor ID of the ECS instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "instance_cpu_core_count" {
  description = "The number of CPU cores of the ECS instance"
  type        = number
  default     = 2
}

variable "instance_memory_size" {
  description = "The memory size in GB of the ECS instance"
  type        = number
  default     = 4
}

variable "instance_image_id" {
  description = "The image ID of the ECS instance"
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
  description = "The operating system of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  performance_type  = var.instance_performance_type
  cpu_core_count    = var.instance_cpu_core_count
  memory_size       = var.instance_memory_size
}

data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}
```

**Parameter Description**:
- **availability_zone**: Assigned by referencing the input variable availability_zone; when empty, the availability zone list in the current region is queried automatically
- **instance_flavor_id**: Assigned by referencing the input variable instance_flavor_id; when empty, the flavor list is queried by performance type, CPU core count, and memory size
- **instance_performance_type**: Assigned by referencing the input variable instance_performance_type, used to filter the flavor performance type
- **instance_cpu_core_count**: Assigned by referencing the input variable instance_cpu_core_count, used to filter the flavor CPU core count
- **instance_memory_size**: Assigned by referencing the input variable instance_memory_size, used to filter the flavor memory size
- **instance_image_id**: Assigned by referencing the input variable instance_image_id; when empty, the image list is queried by flavor, visibility, and OS
- **instance_image_visibility**: Assigned by referencing the input variable instance_image_visibility, used to filter the image visibility
- **instance_image_os**: Assigned by referencing the input variable instance_image_os, used to filter the image OS

### 3. Create a VPC and Subnets

Add the following script in the TF file (such as main.tf) to create a VPC and subnets:

```hcl
# Create a VPC and subnets in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

variable "subnet_configurations" {
  description = "The list of subnet configurations for ECS instance."
  type        = list(object({
    subnet_name       = string
    subnet_cidr       = optional(string)
    subnet_gateway_ip = optional(string)
  }))
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  count = length(var.subnet_configurations)

  vpc_id     = huaweicloud_vpc.test.id
  name       = lookup(var.subnet_configurations[count.index], "subnet_name", null)
  cidr       = try(coalesce(lookup(var.subnet_configurations[count.index], "subnet_cidr", null), cidrsubnet(huaweicloud_vpc.test.cidr, 8, count.index)), null)
  gateway_ip = try(coalesce(lookup(var.subnet_configurations[count.index], "subnet_gateway_ip", null), cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, count.index), 1)), null)
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable vpc_name, used to specify the VPC name
- **cidr**: Assigned by referencing the input variable vpc_cidr, used to specify the VPC CIDR block
- **vpc_id**: Assigned by referencing the ID of the VPC resource
- **subnet_name**: Assigned by referencing subnet_name in the input variable subnet_configurations, used to specify the subnet name
- **subnet_cidr**: Assigned by referencing subnet_cidr in the input variable subnet_configurations; when not specified, it is calculated automatically based on the VPC CIDR block
- **subnet_gateway_ip**: Assigned by referencing subnet_gateway_ip in the input variable subnet_configurations; when not specified, it is calculated automatically based on the subnet CIDR block

### 4. Create a Security Group

Add the following script in the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable security_group_name, used to specify the security group name
- **delete_default_rules**: Set to true to delete the default rules of the security group

### 5. Create an ECS Instance

Add the following script in the TF file (such as main.tf) to create an ECS instance:

```hcl
# Create an ECS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "instance_admin_password" {
  description = "The login password of the ECS instance"
  type        = string
  sensitive   = true
}

resource "huaweicloud_compute_instance" "test" {
  name               = var.instance_name
  image_id           = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
  flavor_id          = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  availability_zone  = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  admin_pass         = var.instance_admin_password

  network {
    uuid = try(huaweicloud_vpc_subnet.test[0].id, null)
  }

  # When using `huaweicloud_compute_interface_attach`, if the security group is not specified, the default security group will be automatically added to the ECS instance.
  lifecycle {
    ignore_changes = [
      security_group_ids
    ]
  }
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable instance_name, used to specify the ECS instance name
- **image_id**: Assigned by referencing the input variable instance_image_id; when not specified, the queried image ID is used
- **flavor_id**: Assigned by referencing the input variable instance_flavor_id; when not specified, the queried flavor ID is used
- **security_group_ids**: Assigned by referencing the ID of the security group resource
- **availability_zone**: Assigned by referencing the input variable availability_zone; when not specified, the queried availability zone name is used
- **admin_pass**: Assigned by referencing the input variable instance_admin_password, used to specify the instance login password
- **network/uuid**: Assigned by referencing the ID of the first subnet resource, used as the primary NIC network of the instance

### 6. Attach the Network Interface

Add the following script in the TF file (such as main.tf) to attach the network interface to the ECS instance:

```hcl
# Attach the network interface to the ECS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "attached_network_id" {
  description = "The ID of the network to which the ECS instance to be attached"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.attached_network_id != "" || length(var.subnet_configurations) == 2
    error_message = "When attached_network_id is not provided, subnet_configurations must have exactly 2 elements."
  }
}

variable "attached_interface_fixed_ip" {
  description = "The fixed IP address of the ECS instance to be attached"
  type        = string
  default     = null
}

variable "attached_security_group_ids" {
  description = "The list of security group IDs of the ECS instance to be attached"
  type        = list(string)
  default     = null
}

resource "huaweicloud_compute_interface_attach" "test" {
  instance_id        = huaweicloud_compute_instance.test.id
  network_id         = var.attached_network_id != "" ? var.attached_network_id : try(huaweicloud_vpc_subnet.test[1].id, null)
  fixed_ip           = var.attached_interface_fixed_ip
  security_group_ids = var.attached_security_group_ids
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the ID of the ECS instance resource
- **network_id**: Assigned by referencing the input variable attached_network_id; when not specified, the ID of the second subnet resource is used
- **fixed_ip**: Assigned by referencing the input variable attached_interface_fixed_ip, used to specify the fixed IP address of the network interface
- **security_group_ids**: Assigned by referencing the input variable attached_security_group_ids, used to specify the security groups of the network interface

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
vpc_name              = "tf_test_ecs_instace"
subnet_configurations = [
  {
    subnet_name = "tf_test_main"
  },
  {
    subnet_name = "tf_test_standby"
  },
]

security_group_name     = "tf_test_ecs_instace"
instance_name           = "tf_test_ecs_instace"
instance_admin_password = "YourPassword!"
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

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the ECS instance with an attached network interface
4. Run `terraform show` to view the created ECS instance with an attached network interface

## Reference Information

- [Huawei Cloud ECS Product Documentation](https://support.huaweicloud.com/ecs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ECS Instance with Network Interface](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ecs/attached-interface)
