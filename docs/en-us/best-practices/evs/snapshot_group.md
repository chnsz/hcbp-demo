# Deploy Disk Snapshot Group

## Application Scenario

Elastic Volume Service (EVS) is a high-performance, highly reliable, and scalable block storage service provided by Huawei Cloud, providing persistent storage for ECS instances. EVS supports multiple storage types, including SSD, SAS, SATA, etc., meeting storage requirements for different business scenarios.

EVS disk snapshot groups are an important feature of the EVS service, used to create consistent snapshot backups of multiple cloud volumes. Through disk snapshot groups, enterprises can ensure data consistency of multiple related cloud volumes at specific points in time, which is very important for scenarios such as database clusters and distributed applications that require data consistency. This best practice will introduce how to use Terraform to automatically deploy EVS disk snapshot groups, including ECS instance creation, cloud volume creation, mounting, and disk snapshot group configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS Flavor List Query Data Source (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [Image List Query Data Source (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS Instance Resource (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [Cloud Volume Resource (huaweicloud_evs_volume)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)
- [Cloud Volume Mount Resource (huaweicloud_compute_volume_attach)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_volume_attach)
- [EVS Disk Snapshot Group Resource (huaweicloud_evs_snapshot_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_snapshot_group)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    ├── data.huaweicloud_compute_flavors.test
    ├── huaweicloud_vpc_subnet.test
    └── huaweicloud_compute_instance.test

data.huaweicloud_compute_flavors.test
    ├── data.huaweicloud_images_images.test
    └── huaweicloud_compute_instance.test

data.huaweicloud_images_images.test
    └── huaweicloud_compute_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test

huaweicloud_vpc_subnet.test
    └── huaweicloud_compute_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_compute_instance.test

huaweicloud_compute_instance.test
    └── huaweicloud_compute_volume_attach.test

huaweicloud_evs_volume.test
    └── huaweicloud_compute_volume_attach.test

huaweicloud_compute_volume_attach.test
    └── huaweicloud_evs_snapshot_group.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zone Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the results of which are used to create ECS instances and cloud volumes:

```hcl
# Get all availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create ECS instances and cloud volumes
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- No additional parameters required, the data source will automatically get all availability zone information in the current region

### 3. Query ECS Flavor Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the results of which are used to create ECS instances:

```hcl
variable "instance_flavor_id" {
  description = "ECS instance flavor ID, if not specified, will use the first available flavor matching the conditions"
  type        = string
  default     = ""
}

variable "availability_zone" {
  description = "ECS instance creation availability zone, if not specified, will use the first availability zone"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "ECS instance flavor performance type, used to query available flavors when instance_flavor_id is not specified"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "ECS instance flavor CPU core count, used to query available flavors when instance_flavor_id is not specified"
  type        = number
  default     = 0
}

variable "instance_flavor_memory_size" {
  description = "ECS instance flavor memory size (GB), used to query available flavors when instance_flavor_id is not specified"
  type        = number
  default     = 0
}

# Get all ECS flavor information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create ECS instances
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**Parameter Description**:
- **availability_zone**: Availability zone, prioritizes using input variable, uses first result from availability zone data source if empty
- **performance_type**: Performance type, assigned by referencing the input variable instance_flavor_performance_type
- **cpu_core_count**: CPU core count, assigned by referencing the input variable instance_flavor_cpu_core_count
- **memory_size**: Memory size, assigned by referencing the input variable instance_flavor_memory_size

### 4. Query Image Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the results of which are used to create ECS instances:

```hcl
variable "instance_image_id" {
  description = "ECS instance image ID, if not specified, will use the first available image matching the conditions"
  type        = string
  default     = ""
}

variable "instance_image_os_type" {
  description = "ECS instance image operating system type, used to query available images when instance_image_id is not specified"
  type        = string
  default     = "Ubuntu"
}

variable "instance_image_visibility" {
  description = "ECS instance image visibility, used to query available images when instance_image_id is not specified"
  type        = string
  default     = "public"
}

# Get all image information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create ECS instances
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].ids[0], "") : var.instance_flavor_id
  os         = var.instance_image_os_type
  visibility = var.instance_image_visibility
}
```

**Parameter Description**:
- **flavor_id**: Flavor ID, prioritizes using input variable, uses first result from ECS flavor data source if empty
- **os**: Operating system type, assigned by referencing the input variable instance_image_os_type
- **visibility**: Visibility, assigned by referencing the input variable instance_image_visibility

### 5. Create VPC

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  description = "VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "192.168.0.0/16"
}

# Create a VPC resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable vpc_cidr

### 6. Create VPC Subnet

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "VPC subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "VPC subnet CIDR block, if not specified, will calculate subnet cidr within existing CIDR address block"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "VPC subnet gateway IP, if not specified, will calculate gateway IP within existing CIDR address block"
  type        = string
  default     = ""
}

# Create a VPC subnet resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  name              = var.subnet_name
  cidr              = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip        = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **vpc_id**: VPC ID, assigned by referencing the VPC resource (huaweicloud_vpc.test) ID
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block, prioritizes using input variable, calculates using cidrsubnet function if empty
- **gateway_ip**: Gateway IP, prioritizes using input variable, calculates using cidrhost function if empty
- **availability_zone**: Availability zone, prioritizes using input variable, uses first result from availability zone data source if empty

### 7. Create Security Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group resource:

```hcl
variable "secgroup_name" {
  description = "Security group name"
  type        = string
}

# Create a security group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.secgroup_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable secgroup_name
- **delete_default_rules**: Delete default rules, set to true

### 8. Create ECS Instance

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an ECS instance resource:

```hcl
variable "ecs_instance_name" {
  description = "ECS instance name"
  type        = string
}

variable "key_pair_name" {
  description = "ECS login key pair name"
  type        = string
  default     = ""
}

variable "system_disk_type" {
  description = "System disk type"
  type        = string
  default     = "SAS"
}

variable "system_disk_size" {
  description = "System disk size (GB)"
  type        = number
  default     = 40
}

# Create an ECS instance resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_compute_instance" "test" {
  name              = var.ecs_instance_name
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  flavor_id         = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, "") : var.instance_flavor_id
  image_id          = var.instance_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, "") : var.instance_image_id
  security_groups   = [huaweicloud_networking_secgroup.test.name]
  key_pair          = var.key_pair_name
  system_disk_type  = var.system_disk_type
  system_disk_size  = var.system_disk_size

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }
}
```

**Parameter Description**:
- **name**: Instance name, assigned by referencing the input variable ecs_instance_name
- **availability_zone**: Availability zone, prioritizes using input variable, uses first result from availability zone data source if empty
- **flavor_id**: Flavor ID, prioritizes using input variable, uses first result from ECS flavor data source if empty
- **image_id**: Image ID, prioritizes using input variable, uses first result from image data source if empty
- **security_groups**: Security group list, referencing security group resource
- **key_pair**: Key pair name, assigned by referencing the input variable key_pair_name
- **system_disk_type**: System disk type, assigned by referencing the input variable system_disk_type
- **system_disk_size**: System disk size, assigned by referencing the input variable system_disk_size

### 9. Create Cloud Volumes

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create cloud volume resources:

```hcl
variable "volume_configuration" {
  description = "List of cloud volume configurations to mount to ECS instance"
  type = list(object({
    name        = string
    size        = number
    volume_type = string
    device_type = string
  }))
  default = []
}

# Create cloud volume resources under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_evs_volume" "test" {
  count = length(var.volume_configuration)

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  volume_type       = var.volume_configuration[count.index].volume_type
  name              = var.volume_configuration[count.index].name
  size              = var.volume_configuration[count.index].size
  device_type       = var.volume_configuration[count.index].device_type
}
```

**Parameter Description**:
- **availability_zone**: Availability zone, prioritizes using input variable, uses first result from availability zone data source if empty
- **volume_type**: Cloud volume type, assigned by referencing the volume_type in input variable volume_configuration
- **name**: Cloud volume name, assigned by referencing the name in input variable volume_configuration
- **size**: Cloud volume size, assigned by referencing the size in input variable volume_configuration
- **device_type**: Device type, assigned by referencing the device_type in input variable volume_configuration

### 10. Mount Cloud Volumes

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create cloud volume mount resources:

```hcl
# Create cloud volume mount resources under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_compute_volume_attach" "test" {
  count = length(var.volume_configuration)

  instance_id = huaweicloud_compute_instance.test.id
  volume_id   = huaweicloud_evs_volume.test[count.index].id
}
```

**Parameter Description**:
- **instance_id**: ECS instance ID, assigned by referencing the ECS instance resource (huaweicloud_compute_instance.test) ID
- **volume_id**: Cloud volume ID, assigned by referencing the cloud volume resource (huaweicloud_evs_volume.test) ID

### 11. Create EVS Disk Snapshot Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an EVS disk snapshot group resource:

```hcl
variable "instant_access" {
  description = "Whether to enable disk snapshot group instant access"
  type        = bool
  default     = false
}

variable "snapshot_group_name" {
  description = "Disk snapshot group name"
  type        = string
  default     = ""
}

variable "snapshot_group_description" {
  description = "Disk snapshot group description"
  type        = string
  default     = "Created by Terraform"
}

variable "enterprise_project_id" {
  description = "Disk snapshot group enterprise project ID"
  type        = string
  default     = "0"
}

variable "incremental" {
  description = "Whether to create incremental snapshots"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Key-value pairs associated with disk snapshot group"
  type        = map(string)
  default = {
    environment = "test"
    created_by  = "terraform"
  }
}

# Create an EVS disk snapshot group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_evs_snapshot_group" "test" {
  depends_on = [huaweicloud_compute_volume_attach.test]

  server_id             = huaweicloud_compute_instance.test.id
  volume_ids            = length(huaweicloud_evs_volume.test) > 0 ? try([for v in huaweicloud_evs_volume.test : v.id], null) : null
  instant_access        = var.instant_access
  name                  = var.snapshot_group_name
  description           = var.snapshot_group_description
  enterprise_project_id = var.enterprise_project_id
  incremental           = var.incremental
  tags                  = var.tags
}
```

**Parameter Description**:
- **server_id**: ECS instance ID, assigned by referencing the ECS instance resource (huaweicloud_compute_instance.test) ID
- **volume_ids**: Cloud volume ID list, assigned by referencing the cloud volume resource list
- **instant_access**: Instant access, assigned by referencing the input variable instant_access
- **name**: Disk snapshot group name, assigned by referencing the input variable snapshot_group_name
- **description**: Disk snapshot group description, assigned by referencing the input variable snapshot_group_description
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **incremental**: Incremental snapshot, assigned by referencing the input variable incremental
- **tags**: Tags, assigned by referencing the input variable tags

### 12. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Network configuration
vpc_name    = "evs-test-vpc"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "evs-test-subnet"
secgroup_name = "evs-test-sg"

# ECS instance configuration
ecs_instance_name = "evs-test-ecs"

# Cloud volume configuration
volume_configuration = [
  {
    name        = "evs-test-volume1"
    size        = 50
    volume_type = "SSD"
    device_type = "VBD"
  },
  {
    name        = "evs-test-volume2"
    size        = 100
    volume_type = "SAS"
    device_type = "SCSI"
  }
]
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="ecs_instance_name=my-ecs"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 13. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating disk snapshot groups
4. Run `terraform show` to view the created disk snapshot groups

## Reference Information

- [Huawei Cloud EVS Product Documentation](https://support.huaweicloud.com/evs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EVS Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/evs)
