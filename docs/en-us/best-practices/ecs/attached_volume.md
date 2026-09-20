# Deploy Instance with Volume

## Application Scenario

Elastic Cloud Server (ECS) is a fundamental computing component composed of CPU, memory, operating system, and cloud disks. In real-world business scenarios, the system disk is usually only used to host the operating system, while databases, logs, and application data require independent and scalable persistent storage space, which means additional data volumes (EVS disks) need to be attached to the ECS instance.

This best practice will introduce how to use Terraform to automatically deploy an ECS instance with an attached data volume, including automatic querying of availability zones, flavors, and images, creation of VPC, subnet, and security group, creation of the ECS instance, and creation of the EVS volume attached to the instance.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS Flavors (huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS Images (huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS Instance (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [EVS Volume (huaweicloud_evs_volume)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── data.huaweicloud_images_images
            └── huaweicloud_compute_instance
                └── huaweicloud_evs_volume

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones, Flavors, and Images

Add the following script to the TF file (such as main.tf) to query the availability zones, flavors, and images required for deploying the ECS instance:

```hcl
# Query the availability zone list in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone to which the ECS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

# Query the ECS flavor list in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  performance_type  = var.instance_performance_type
  cpu_core_count    = var.instance_cpu_core_count
  memory_size       = var.instance_memory_size
}

# Query the IMS image list in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}
```

**Parameter Description**:
- **count**: Controls whether the data source is queried through a conditional expression, skipping the query when the corresponding value is already specified by the input variable
- **availability_zone**: Assigned by referencing the input variable availability_zone, taking the first availability zone in the list when not specified
- **performance_type**: Assigned by referencing the input variable instance_performance_type, used to filter the performance type of the flavor
- **cpu_core_count**: Assigned by referencing the input variable instance_cpu_core_count, used to filter the CPU core count of the flavor
- **memory_size**: Assigned by referencing the input variable instance_memory_size, used to filter the memory size of the flavor
- **flavor_id**: Assigned by referencing the input variable instance_flavor_id, taking the first flavor in the list when not specified
- **visibility**: Assigned by referencing the input variable instance_image_visibility, used to filter the visibility of the image
- **os**: Assigned by referencing the input variable instance_image_os, used to filter the operating system of the image

### 3. Create VPC, Subnet, and Security Group

Add the following script to the TF file (such as main.tf) to create the network environment:

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

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

# Create a VPC subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}

# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_ids" {
  description = "The list of security group IDs of the ECS instance"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "security_group_name" {
  description = "The name of the security group"
  type        = string
  default     = ""

  validation {
    condition     = !(length(var.security_group_ids) < 1 && var.security_group_name == "")
    error_message = "The security group name cannot be empty if the security group ID list is not set"
  }
}

resource "huaweicloud_networking_secgroup" "test" {
  count = length(var.security_group_ids) < 1 ? 1 : 0

  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC resource
- **cidr**: The subnet CIDR block, assigned by referencing the input variable subnet_cidr, automatically divided based on the VPC CIDR block when not specified
- **gateway_ip**: The subnet gateway IP, assigned by referencing the input variable subnet_gateway_ip, automatically calculated based on the subnet CIDR block when not specified
- **count**: Controls whether the security group is created through a conditional expression, skipping creation when the security group ID list is already specified
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group, set to true to remove the default allow rules

### 4. Create ECS Instance

Add the following script to the TF file (such as main.tf) to create the ECS instance:

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

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = null
}

resource "huaweicloud_compute_instance" "test" {
  name                  = var.instance_name
  image_id              = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
  flavor_id             = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  security_group_ids    = length(var.security_group_ids) > 0 ? var.security_group_ids : huaweicloud_networking_secgroup.test[*].id
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  admin_pass            = var.instance_admin_password
  enterprise_project_id = var.enterprise_project_id

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }
}
```

**Parameter Description**:
- **name**: The ECS instance name, assigned by referencing the input variable instance_name
- **image_id**: The instance image ID, assigned by referencing the input variable instance_image_id, taking the first image in the list when not specified
- **flavor_id**: The instance flavor ID, assigned by referencing the input variable instance_flavor_id, taking the first flavor in the list when not specified
- **security_group_ids**: The list of security group IDs of the instance, assigned by referencing the input variable security_group_ids, using the created security group when not specified
- **availability_zone**: The availability zone to which the instance belongs, assigned by referencing the input variable availability_zone, taking the first availability zone in the list when not specified
- **admin_pass**: The login password of the instance, assigned by referencing the input variable instance_admin_password
- **enterprise_project_id**: The ID of the enterprise project to which the instance belongs, assigned by referencing the input variable enterprise_project_id
- **network/uuid**: The ID of the subnet associated with the instance network, assigned by referencing the ID of the subnet resource

### 5. Create EVS Volume and Attach to ECS Instance

Add the following script to the TF file (such as main.tf) to create the EVS volume and attach it to the ECS instance:

```hcl
# Create an EVS volume and attach it to the ECS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "volume_name" {
  description = "The name of the data volume"
  type        = string
}

variable "volume_type" {
  description = "The type of the data volume"
  type        = string
  default     = "SSD"
}

variable "volume_size" {
  description = "The size of the data volume in GB"
  type        = number
  default     = 10
}

variable "volume_iops" {
  description = "The IOPS(Input/Output Operations Per Second) for the data volume"
  type        = number
  default     = null
}

variable "volume_throughput" {
  description = "The throughput for the data volume"
  type        = number
  default     = null
}

variable "volume_backup_id" {
  description = "The backup ID from which to create the disk"
  type        = string
  default     = null
}

variable "volume_snapshot_id" {
  description = "The snapshot ID from which to create the disk"
  type        = string
  default     = null
}

resource "huaweicloud_evs_volume" "test" {
  server_id             = huaweicloud_compute_instance.test.id
  name                  = var.volume_name
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  volume_type           = var.volume_type
  size                  = var.volume_size
  iops                  = var.volume_iops
  throughput            = var.volume_throughput
  backup_id             = var.volume_backup_id
  snapshot_id           = var.volume_snapshot_id
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **server_id**: The ID of the target ECS instance to which the volume is attached, assigned by referencing the ID of the ECS instance resource
- **name**: The volume name, assigned by referencing the input variable volume_name
- **availability_zone**: The availability zone to which the volume belongs, assigned by referencing the input variable availability_zone, taking the first availability zone in the list when not specified
- **volume_type**: The volume type, assigned by referencing the input variable volume_type
- **size**: The volume size in GB, assigned by referencing the input variable volume_size
- **iops**: The volume IOPS, assigned by referencing the input variable volume_iops, required when the type is GPSSD2 or ESSD2
- **throughput**: The volume throughput, assigned by referencing the input variable volume_throughput, required when the type is GPSSD2
- **backup_id**: The backup ID from which to create the volume, assigned by referencing the input variable volume_backup_id
- **snapshot_id**: The snapshot ID from which to create the volume, assigned by referencing the input variable volume_snapshot_id
- **enterprise_project_id**: The ID of the enterprise project to which the volume belongs, assigned by referencing the input variable enterprise_project_id

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name                = "tf_test_ecs_instace"
subnet_name             = "tf_test_ecs_instace"
security_group_name     = "tf_test_ecs_instace"
instance_name           = "tf_test_ecs_instace"
instance_admin_password = "YourPassword!"
volume_name             = "tf_test_volume"
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

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the ECS instance with the attached volume
4. Run `terraform show` to view the created ECS instance with the attached volume

## Reference Information

- [Huawei Cloud ECS Product Documentation](https://support.huaweicloud.com/ecs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ECS Instance with Volume](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ecs/attached-volume)
