# Deploy Volume and V5 Snapshot

## Application Scenario

Elastic Volume Service (EVS) provides persistent block storage for Elastic Cloud Servers and is the foundation for business data persistence. In real-world scenarios, besides creating a volume and attaching it to a cloud server, you also need to create snapshots for the volume so that data can be quickly rolled back when data is accidentally deleted, a virus intrudes, or a system failure occurs. Huawei Cloud EVS provides V5 snapshot capabilities, including instant access, incremental snapshots, tags, and metadata, meeting the requirements of different data protection policies.

This best practice will introduce how to use Terraform to automatically deploy a volume and a V5 snapshot, including VPC and subnet creation, security group creation, ECS instance creation, volume creation and metadata configuration, volume attachment, and V5 snapshot creation and metadata configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Compute Flavors (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [Images (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS Instance (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [Volume (huaweicloud_evs_volume)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)
- [Volume Metadata (huaweicloud_evs_volume_metadata)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume_metadata)
- [Volume Attachment (huaweicloud_compute_volume_attach)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_volume_attach)
- [V5 Volume Snapshot (huaweicloud_evsv5_snapshot)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evsv5_snapshot)
- [Volume Snapshot Metadata (huaweicloud_evs_snapshot_metadata)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_snapshot_metadata)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_compute_flavors
    │       └── data.huaweicloud_images_images
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_compute_instance
            └── huaweicloud_compute_volume_attach

data.huaweicloud_compute_flavors
    └── huaweicloud_compute_instance

data.huaweicloud_images_images
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance

huaweicloud_evs_volume
    ├── huaweicloud_evs_volume_metadata
    ├── huaweicloud_compute_volume_attach
    └── huaweicloud_evsv5_snapshot
            └── huaweicloud_evs_snapshot_metadata
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf):

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- This data source requires no additional parameters. It is used to obtain the list of available zones in the current region for subsequent subnet, ECS instance, and volume availability zone selection.

### 3. Query Compute Flavors

Add the following script in the TF file (such as main.tf):

```hcl
# Query the compute flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_flavor_id" {
  description = "The flavor ID of the ECS instance, if not specified, the first available flavor matching the criteria will be used."
  type        = string
  default     = ""
}

variable "availability_zone" {
  description = "The availability zone where the ECS instance will be created, if not specified, the first available zone will be used"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "The number of CPU cores for the ECS instance flavor"
  type        = number
  default     = 0
}

variable "instance_flavor_memory_size" {
  description = "The memory size in GB for the ECS instance flavor"
  type        = number
  default     = 0
}

data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**Parameter Description**:
- **count**: Assigned by referencing the input variable instance_flavor_id; the data source is created when no flavor ID is specified
- **availability_zone**: Assigned by referencing the input variable availability_zone; the first availability zone is used when not specified
- **performance_type**: Assigned by referencing the input variable instance_flavor_performance_type
- **cpu_core_count**: Assigned by referencing the input variable instance_flavor_cpu_core_count
- **memory_size**: Assigned by referencing the input variable instance_flavor_memory_size

### 4. Query Images

Add the following script in the TF file (such as main.tf):

```hcl
# Query the images in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_image_id" {
  description = "The ID of the image used to create the ECS instance, if not specified, the first available image matching the criteria will be used."
  type        = string
  default     = ""
}

variable "instance_image_os_type" {
  description = "The OS type of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

variable "instance_image_visibility" {
  description = "The visibility of the ECS instance image"
  type        = string
  default     = "public"
}

data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].ids[0], "") : var.instance_flavor_id
  os         = var.instance_image_os_type
  visibility = var.instance_image_visibility
}
```

**Parameter Description**:
- **count**: Assigned by referencing the input variable instance_image_id; the data source is created when no image ID is specified
- **flavor_id**: Assigned by referencing the input variable instance_flavor_id; the first flavor ID is used when not specified
- **os**: Assigned by referencing the input variable instance_image_os_type
- **visibility**: Assigned by referencing the input variable instance_image_visibility

### 5. Create a VPC

Add the following script in the TF file (such as main.tf):

```hcl
# Create a VPC in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr

### 6. Create a Subnet

Add the following script in the TF file (such as main.tf):

```hcl
# Create a subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet, if not specified, calculating a subnet cidr within the existing CIDR address block"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet, if not specified, calculating a gateway IP within the existing CIDR address block"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  name              = var.subnet_name
  cidr              = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip        = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing the ID of the VPC resource
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; a subnet CIDR is automatically calculated within the VPC CIDR when not specified
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip; the gateway IP is automatically calculated when not specified
- **availability_zone**: Assigned by referencing the input variable availability_zone; the first availability zone is used when not specified

### 7. Create a Security Group

Add the following script in the TF file (such as main.tf):

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "secgroup_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.secgroup_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable secgroup_name
- **delete_default_rules**: Set to true to delete the default rules of the security group

### 8. Create an ECS Instance

Add the following script in the TF file (such as main.tf):

```hcl
# Create an ECS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "ecs_instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "key_pair_name" {
  description = "The name of the key pair for ECS login"
  type        = string
  default     = ""
}

variable "system_disk_type" {
  description = "The type of the system disk"
  type        = string
  default     = "SAS"
}

variable "system_disk_size" {
  description = "The size of the system disk in GB"
  type        = number
  default     = 40
}

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
- **name**: Assigned by referencing the input variable ecs_instance_name
- **availability_zone**: Assigned by referencing the input variable availability_zone; the first availability zone is used when not specified
- **flavor_id**: Assigned by referencing the input variable instance_flavor_id; the first flavor ID is used when not specified
- **image_id**: Assigned by referencing the input variable instance_image_id; the first image ID is used when not specified
- **security_groups**: Assigned by referencing the name of the security group resource
- **key_pair**: Assigned by referencing the input variable key_pair_name
- **system_disk_type**: Assigned by referencing the input variable system_disk_type
- **system_disk_size**: Assigned by referencing the input variable system_disk_size
- **network.uuid**: Assigned by referencing the ID of the subnet resource

### 9. Create a Volume

Add the following script in the TF file (such as main.tf):

```hcl
# Create a volume in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "volume_availability_zone" {
  description = "The availability zone for the volume"
  type        = string
  default     = ""
  nullable    = false
}

variable "volume_name" {
  description = "The name of the volume"
  type        = string
}

variable "volume_type" {
  description = "The type of the volume"
  type        = string
  default     = "SSD"
}

variable "volume_size" {
  description = "The size of the volume"
  type        = number
  default     = 40
}

variable "volume_description" {
  description = "The description of the volume"
  type        = string
  default     = ""
}

variable "volume_multiattach" {
  description = "The volume is shared volume or not"
  type        = bool
  default     = false
}

variable "volume_iops" {
  description = "The IOPS for the volume"
  type        = number
  default     = null
}

variable "volume_throughput" {
  description = "The throughput for the volume"
  type        = number
  default     = null
}

variable "volume_device_type" {
  description = "The device type of disk to create"
  type        = string
  default     = "VBD"
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = null
}

variable "volume_tags" {
  description = "The tags of the volume"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the volume"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the volume"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the volume"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the volume"
  type        = string
  default     = "false"
}

resource "huaweicloud_evs_volume" "test" {
  availability_zone     = var.volume_availability_zone != "" ? var.volume_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  name                  = var.volume_name
  volume_type           = var.volume_type
  size                  = var.volume_size
  description           = var.volume_description
  multiattach           = var.volume_multiattach
  iops                  = var.volume_iops
  throughput            = var.volume_throughput
  device_type           = var.volume_device_type
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.volume_tags
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew
}
```

**Parameter Description**:
- **availability_zone**: Assigned by referencing the input variable volume_availability_zone; the first availability zone is used when not specified
- **name**: Assigned by referencing the input variable volume_name
- **volume_type**: Assigned by referencing the input variable volume_type
- **size**: Assigned by referencing the input variable volume_size
- **description**: Assigned by referencing the input variable volume_description
- **multiattach**: Assigned by referencing the input variable volume_multiattach
- **iops**: Assigned by referencing the input variable volume_iops; valid only when the volume type is GPSSD2 or ESSD2
- **throughput**: Assigned by referencing the input variable volume_throughput; valid only when the volume type is GPSSD2
- **device_type**: Assigned by referencing the input variable volume_device_type
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id
- **tags**: Assigned by referencing the input variable volume_tags
- **charging_mode**: Assigned by referencing the input variable charging_mode
- **period_unit**: Assigned by referencing the input variable period_unit; valid only when the charging mode is prePaid
- **period**: Assigned by referencing the input variable period; valid only when the charging mode is prePaid
- **auto_renew**: Assigned by referencing the input variable auto_renew

### 10. Create Volume Metadata

Add the following script in the TF file (such as main.tf):

```hcl
# Create volume metadata in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "volume_metadata" {
  description = "The volume metadata"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_evs_volume_metadata" "test" {
  volume_id = huaweicloud_evs_volume.test.id
  metadata  = var.volume_metadata
}
```

**Parameter Description**:
- **volume_id**: Assigned by referencing the ID of the volume resource
- **metadata**: Assigned by referencing the input variable volume_metadata

### 11. Attach the Volume to the ECS Instance

Add the following script in the TF file (such as main.tf):

```hcl
# Attach the volume to the ECS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_compute_volume_attach" "test" {
  instance_id = huaweicloud_compute_instance.test.id
  volume_id   = huaweicloud_evs_volume.test.id
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the ID of the ECS instance resource
- **volume_id**: Assigned by referencing the ID of the volume resource

### 12. Create a V5 Volume Snapshot

Add the following script in the TF file (such as main.tf):

```hcl
# Create a V5 volume snapshot in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "snapshot_name" {
  description = "The name of the snapshot"
  type        = string
}

variable "snapshot_description" {
  description = "The description of the snapshot"
  type        = string
  default     = ""
}

variable "snapshot_tags" {
  description = "The tags of the snapshot"
  type        = map(string)
  default     = {}
}

variable "snapshot_instant_access" {
  description = "Whether to enable instant access for the snapshot"
  type        = bool
  default     = false
}

variable "snapshot_incremental" {
  description = "Whether to create an incremental snapshot"
  type        = bool
  default     = false
}

resource "huaweicloud_evsv5_snapshot" "test" {
  volume_id             = huaweicloud_evs_volume.test.id
  name                  = var.snapshot_name
  description           = var.snapshot_description
  tags                  = var.snapshot_tags
  enterprise_project_id = var.enterprise_project_id
  instant_access        = var.snapshot_instant_access
  incremental           = var.snapshot_incremental
}
```

**Parameter Description**:
- **volume_id**: Assigned by referencing the ID of the volume resource
- **name**: Assigned by referencing the input variable snapshot_name
- **description**: Assigned by referencing the input variable snapshot_description
- **tags**: Assigned by referencing the input variable snapshot_tags
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id
- **instant_access**: Assigned by referencing the input variable snapshot_instant_access
- **incremental**: Assigned by referencing the input variable snapshot_incremental

### 13. Create Volume Snapshot Metadata

Add the following script in the TF file (such as main.tf):

```hcl
# Create volume snapshot metadata in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "snapshot_metadata" {
  description = "The snapshot metadata"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_evs_snapshot_metadata" "test" {
  snapshot_id = huaweicloud_evsv5_snapshot.test.id
  metadata    = var.snapshot_metadata
}
```

**Parameter Description**:
- **snapshot_id**: Assigned by referencing the ID of the V5 volume snapshot resource
- **metadata**: Assigned by referencing the input variable snapshot_metadata

### 14. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication parameters
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Network and ECS parameters
vpc_name          = "tf_test_evs_vpc"
subnet_name       = "tf_test_evs_subnet"
secgroup_name     = "tf_test_evs_sg"
ecs_instance_name = "tf_test_evs_ecs"

# Volume parameters
volume_name        = "tf_test_evs_volume"
volume_description = "terraform volume test"

volume_tags = {
  foo = "bar"
  key = "value"
}

volume_metadata = {
  metadata_key = "volume"
  key          = "value"
}

# V5 snapshot parameters
snapshot_name        = "tf_test_evs_v5_snapshot"
snapshot_description = "terraform v5 snapshot test"

snapshot_tags = {
  foo = "bar"
  key = "value"
}

snapshot_metadata = {
  metadata_key = "snapshot"
  key          = "value"
}
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

### 15. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the volume and V5 snapshot
4. Run `terraform show` to view the created volume and V5 snapshot

## Reference Information

- [Huawei Cloud EVS Product Documentation](https://support.huaweicloud.com/evs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EVS Volume and V5 Snapshot](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/evs/volume-v5-snapshot)
