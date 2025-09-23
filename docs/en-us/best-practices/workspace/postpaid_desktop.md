# Deploy Pay-per-Use Cloud Desktop

## Application Scenario

Huawei Cloud Cloud Desktop (Workspace) is a cloud computing-based desktop virtualization service that provides enterprise users with secure and convenient cloud office solutions.
Cloud desktop provides remote desktop access capabilities, allowing users to access their cloud office environment anytime, anywhere through various terminal devices, while centrally managing data and applications to improve security and work efficiency.
The pay-per-use billing mode allows enterprises to pay flexibly based on actual usage without prepaying large amounts of funds, suitable for temporary projects or scenarios with fluctuating usage.
This best practice will introduce how to use Terraform to automatically deploy pay-per-use cloud desktop instances.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zone List Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Cloud Desktop Flavor List Query Data Source (data.huaweicloud_workspace_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_flavors)
- [IMS Image List Query Data Source (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)
- [Cloud Desktop Service Query Data Source (data.huaweicloud_workspace_service)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_service)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Cloud Desktop Service Resource (huaweicloud_workspace_service)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_service)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule Resource (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Cloud Desktop User Resource (huaweicloud_workspace_user)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_user)
- [Cloud Desktop Instance Resource (huaweicloud_workspace_desktop)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_desktop)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_workspace_flavors
        └── huaweicloud_workspace_desktop

data.huaweicloud_images_images
    └── huaweicloud_workspace_desktop

data.huaweicloud_workspace_service
    ├── huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    ├── huaweicloud_workspace_service
    ├── huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_workspace_desktop

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_workspace_service

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_workspace_desktop

huaweicloud_workspace_service
    └── huaweicloud_workspace_desktop

huaweicloud_workspace_user
    └── huaweicloud_workspace_desktop
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zones Required for Cloud Desktop Instance Resource Creation Through Data Source (data.huaweicloud_availability_zones)

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the results of which are used to create cloud desktop instances:

```hcl
variable "availability_zone" {
  description = "The availability zone to which the cloud desktop flavor and network belong"
  type        = string
  default     = ""
}

# Get all availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create cloud desktop instances
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: Creation count of the data source, used to control whether to execute the availability zone list query data source, only creates the data source when `var.availability_zone` is empty (i.e., executes availability zone list query)

### 3. Query Cloud Desktop Flavors Required for Cloud Desktop Instance Resource Creation Through Data Source (data.huaweicloud_workspace_flavors)

Add the following script to the TF file to instruct Terraform to query cloud desktop flavors that meet the conditions:

```hcl
variable "desktop_flavor_id" {
  description = "The flavor ID of the cloud desktop"
  type        = string
  default     = ""
}

variable "desktop_flavor_os_type" {
  description = "The OS type of the cloud desktop flavor"
  type        = string
  default     = "Windows"
}

variable "desktop_flavor_cpu_core_number" {
  description = "The number of the cloud desktop flavor CPU cores"
  type        = number
  default     = 4
}

variable "desktop_flavor_memory_size" {
  description = "The number of the cloud desktop flavor memories"
  type        = number
  default     = 8
}

# Get all cloud desktop flavor information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) that meets specific conditions, used to create cloud desktop instances
data "huaweicloud_workspace_flavors" "test" {
  count = var.desktop_flavor_id == "" ? 1 : 0

  os_type           = var.desktop_flavor_os_type
  vcpus             = var.desktop_flavor_cpu_core_number
  memory            = var.desktop_flavor_memory_size
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **count**: Creation count of the data source, used to control whether to execute the cloud desktop flavor list query data source, only creates the data source when `var.desktop_flavor_id` is empty (i.e., executes cloud desktop flavor list query)
- **os_type**: Operating system type, optional values: Windows, Linux
- **vcpus**: CPU core count, used to filter flavors
- **memory**: Memory size (GB), used to filter flavors
- **availability_zone**: Availability zone where the flavor is located, prioritizes using the availability zone specified in input variables, uses the first availability zone from data source query if not specified

### 4. Query Cloud Desktop Images Required for Cloud Desktop Instance Resource Creation Through Data Source (data.huaweicloud_images_images)

Add the following script to the TF file to instruct Terraform to query cloud desktop images that meet the conditions:

```hcl
variable "desktop_image_id" {
  description = "The specified image ID that the cloud desktop used"
  type        = string
  default     = ""
}

variable "desktop_image_os_type" {
  description = "The OS type of the cloud desktop image"
  type        = string
  default     = "Windows"
}

variable "desktop_image_visibility" {
  description = "The visibility of the cloud desktop image"
  type        = string
  default     = "market"
}

# Get all cloud desktop image information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) that meets specific conditions, used to create cloud desktop instances
data "huaweicloud_images_images" "test" {
  count = var.desktop_image_id == "" ? 1 : 0

  name_regex = "WORKSPACE"
  os         = var.desktop_image_os_type
  visibility = var.desktop_image_visibility
}
```

**Parameter Description**:
- **count**: Creation count of the data source, used to control whether to execute the image list query data source, only creates the data source when `var.desktop_image_id` is empty (i.e., executes image list query)
- **name_regex**: Regular expression for image names, used to filter cloud desktop related images
- **os**: Operating system type, used to filter images
- **visibility**: Image visibility, market indicates cloud market images

### 5. Query Cloud Desktop Service Status Through Data Source (data.huaweicloud_workspace_service)

Add the following script to the TF file to instruct Terraform to query the current cloud desktop service status:

```hcl
# Get cloud desktop service status information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to determine whether to create related network resources
data "huaweicloud_workspace_service" "test" {}
```

**Parameter Description**:
This data source is used to query the current cloud desktop service status. If the service status is "CLOSED", VPC, subnet, security group, and other network resources need to be created; if the service is already enabled, existing network resources can be reused.

### 6. Create VPC Resource (huaweicloud_vpc)

Add the following script to the TF file to instruct Terraform to conditionally create VPC resources based on cloud desktop service status:

```hcl
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

# Create a VPC resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) conditionally based on cloud desktop service status, used to deploy cloud desktop instances
resource "huaweicloud_vpc" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **count**: Creation count of the resource, only creates VPC resource when cloud desktop service status is "CLOSED"
- **name**: VPC name, assigned by referencing the input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable vpc_cidr

### 7. Create VPC Subnet Resource (huaweicloud_vpc_subnet)

Add the following script to the TF file to instruct Terraform to conditionally create VPC subnet resources based on cloud desktop service status:

```hcl
variable "subnet_name" {
  description = "The subnet name"
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

# Create a VPC subnet resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) conditionally based on cloud desktop service status, used to deploy cloud desktop instances
resource "huaweicloud_vpc_subnet" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  vpc_id     = try(huaweicloud_vpc.test[0].id, null)
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(try(huaweicloud_vpc.test[0].cidr, "192.168.0.0/16"), 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(try(huaweicloud_vpc.test[0].cidr, "192.168.0.0/16"), 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **count**: Creation count of the resource, only creates subnet resource when cloud desktop service status is "CLOSED"
- **vpc_id**: VPC ID that the subnet belongs to, referencing the ID of the previously created VPC resource
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block, uses cidrsubnet function to divide a subnet segment from VPC's CIDR block if not specified
- **gateway_ip**: Subnet gateway IP, uses cidrhost function to get the first IP address from subnet segment as gateway IP if not specified

### 8. Create Cloud Desktop Service (huaweicloud_workspace_service)

Add the following script to the TF file to instruct Terraform to conditionally create cloud desktop service resources based on cloud desktop service status:

```hcl
# Create a cloud desktop service resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) conditionally based on cloud desktop service status, used to deploy cloud desktop instances
resource "huaweicloud_workspace_service" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  access_mode = "INTERNET"
  vpc_id      = try(huaweicloud_vpc.test[0].id, null)
  network_ids = [
    try(huaweicloud_vpc_subnet.test[0].id, null),
  ]
}
```

**Parameter Description**:
- **count**: Creation count of the resource, only creates cloud desktop service resource when cloud desktop service status is "CLOSED"
- **access_mode**: Access mode, using INTERNET indicates access through public network
- **vpc_id**: VPC ID, referencing the ID of the previously created VPC resource
- **network_ids**: Network ID list, referencing the ID of the previously created subnet resource

### 9. Create Security Group Resource (huaweicloud_networking_secgroup)

Add the following script to the TF file to instruct Terraform to conditionally create security group resources based on cloud desktop service status:

```hcl
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

# Create a security group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) conditionally based on cloud desktop service status, used to deploy cloud desktop instances
resource "huaweicloud_networking_secgroup" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **count**: Creation count of the resource, only creates security group resource when cloud desktop service status is "CLOSED"
- **name**: Security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to true to delete default rules

### 10. Create Security Group Rule Resource (huaweicloud_networking_secgroup_rule)

Add the following script to the TF file to instruct Terraform to conditionally create security group rule resources based on cloud desktop service status:

```hcl
# Create a security group rule resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) conditionally based on cloud desktop service status, used to deploy cloud desktop instances
resource "huaweicloud_networking_secgroup_rule" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  security_group_id = try(huaweicloud_networking_secgroup.test[0].id, null)
  direction         = "egress"
  ethertype         = "IPv4"
  remote_ip_prefix  = "0.0.0.0/0"
  priority          = 1
}
```

**Parameter Description**:
- **count**: Creation count of the resource, only creates security group rule resource when cloud desktop service status is "CLOSED"
- **security_group_id**: Security group ID, referencing the ID of the previously created security group resource
- **direction**: Rule direction, egress indicates outbound traffic
- **ethertype**: IP protocol version, IPv4 indicates IPv4 protocol
- **remote_ip_prefix**: Remote IP address, 0.0.0.0/0 indicates allowing all IP addresses
- **priority**: Rule priority, smaller values have higher priority

### 11. Create Cloud Desktop User (huaweicloud_workspace_user)

Add the following script to the TF file to instruct Terraform to create a cloud desktop user resource:

```hcl
variable "desktop_user_name" {
  description = "The user name that the cloud desktop used"
  type        = string
}

variable "desktop_user_email" {
  description = "The email address that the user used"
  type        = string
}

# Create a cloud desktop user resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to deploy cloud desktop instances
resource "huaweicloud_workspace_user" "test" {
  depends_on = [huaweicloud_workspace_service.test]

  name  = var.desktop_user_name
  email = var.desktop_user_email

  account_expires            = "0"
  password_never_expires     = false
  enable_change_password     = true
  next_login_change_password = true
  disabled                   = false
}
```

**Parameter Description**:
- **name**: Username, assigned by referencing the input variable desktop_user_name
- **email**: User email, assigned by referencing the input variable desktop_user_email
- **account_expires**: Account expiration time, set to "0" indicates never expires
- **password_never_expires**: Whether password never expires, set to false indicates password has expiration time
- **enable_change_password**: Whether to allow password change, set to true indicates allowing password change
- **next_login_change_password**: Whether to change password on next login, set to true indicates password change required on next login
- **disabled**: Whether to disable user, set to false indicates user is not disabled

### 12. Create Cloud Desktop Instance (huaweicloud_workspace_desktop)

Add the following script to the TF file to instruct Terraform to create a cloud desktop instance resource:

```hcl
variable "cloud_desktop_name" {
  description = "The cloud desktop name"
  type        = string
}

variable "desktop_user_group_name" {
  description = "The name of the user group that cloud desktop used"
  type        = string
  default     = "users"
}

variable "desktop_root_volume_type" {
  description = "The storage type of system disk"
  type        = string
  default     = "SSD"
}

variable "desktop_root_volume_size" {
  description = "The storage capacity of system disk"
  type        = number
  default     = 100
}

variable "desktop_data_volumes" {
  description = "The storage configuration of data disks"
  type = list(object({
    type = string
    size = number
  }))
  default = [
    {
      type = "SSD",
      size = 100,
    },
  ]
}

# Create a cloud desktop instance resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_workspace_desktop" "test" {
  depends_on = [huaweicloud_workspace_user.test]

  flavor_id         = var.desktop_flavor_id == "" ? try([for o in data.huaweicloud_workspace_flavors.test[0].flavors: o.id if !strcontains(lower(o.description), "flexus")][0], null) : var.desktop_flavor_id
  image_type        = var.desktop_image_visibility
  image_id          = var.desktop_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, null) : var.desktop_image_id
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id            = data.huaweicloud_workspace_service.test.status != "CLOSED" ? data.huaweicloud_workspace_service.test.vpc_id : try(huaweicloud_vpc.test[0].id, null)
  security_groups   = data.huaweicloud_workspace_service.test.status != "CLOSED" ? concat(
    data.huaweicloud_workspace_service.test.desktop_security_group[*].id,
    data.huaweicloud_workspace_service.test.infrastructure_security_group[*].id,
    try(huaweicloud_networking_secgroup.test[0].id, []),
  ) : concat(
    try(huaweicloud_workspace_service.test[0].desktop_security_group[*].id, []),
    try(huaweicloud_workspace_service.test[0].infrastructure_security_group[*].id, []),
    try(huaweicloud_networking_secgroup.test[0].id, []),
  )

  dynamic "nic" {
    for_each = data.huaweicloud_workspace_service.test.status != "CLOSED" ? data.huaweicloud_workspace_service.test.network_ids : try([huaweicloud_vpc_subnet.test[0].id], [])

    content {
      network_id = nic.value
    }
  }

  name       = var.cloud_desktop_name
  user_name  = huaweicloud_workspace_user.test.name
  user_email = huaweicloud_workspace_user.test.email
  user_group = var.desktop_user_group_name

  root_volume {
    type = var.desktop_root_volume_type
    size = var.desktop_root_volume_size
  }

  dynamic "data_volume" {
    for_each = var.desktop_data_volumes

    content {
      type = data_volume.value["type"]
      size = data_volume.value["size"]
    }
  }

  lifecycle {
    ignore_changes = [
      flavor_id,
      image_id,
      availability_zone,
    ]
  }
}
```

**Parameter Description**:
- **flavor_id**: Cloud desktop flavor ID, prioritizes using the flavor specified in input variables, uses the first non-flexus flavor from data source query if not specified
- **image_type**: Image type, assigned by referencing the input variable desktop_image_visibility
- **image_id**: Image ID, prioritizes using the image specified in input variables, uses the first image from data source query if not specified
- **availability_zone**: Availability zone, prioritizes using the availability zone specified in input variables, uses the first availability zone from data source query if not specified
- **vpc_id**: VPC ID, uses existing VPC or newly created VPC based on cloud desktop service status
- **security_groups**: Security group ID list, uses different security group configurations based on cloud desktop service status
- **nic**: Network interface configuration block (dynamic block), uses different network configurations based on cloud desktop service status
  - **network_id**: Unique identifier of the network, uses existing network or newly created subnet based on service status
- **name**: Cloud desktop name, assigned by referencing the input variable cloud_desktop_name
- **user_name**: Username, referencing the name of the previously created cloud desktop user resource
- **user_email**: User email, referencing the email of the previously created cloud desktop user resource
- **user_group**: User group name, assigned by referencing the input variable desktop_user_group_name, default is "users"
- **root_volume**: System disk configuration block
  - **type**: Disk type, assigned by referencing the input variable desktop_root_volume_type, default is SSD
  - **size**: Disk size, assigned by referencing the input variable desktop_root_volume_size, default is 100GB
- **data_volume**: Data disk configuration block (dynamic block)
  - **type**: Disk type, assigned by referencing the type value in the input variable desktop_data_volumes
  - **size**: Disk size, assigned by referencing the size value in the input variable desktop_data_volumes
- **lifecycle**: Lifecycle management, ignores changes to flavor, image, and availability zone to avoid instance recreation

### 13. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following content:

```hcl
vpc_name            = "tf_test_vpc"
vpc_cidr            = "192.168.0.0/16"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
desktop_user_name   = "tf_test_user"
desktop_user_email  = "test@example.com"
cloud_desktop_name  = "tf-test-desktop"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

> For variables not specified in the `terraform.tfvars` file, Terraform will use the default values defined in the code or prompt the user for input during execution.

### 14. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating cloud desktop instances
4. Run `terraform show` to view the created cloud desktop instance details

## Reference Information

- [Huawei Cloud Cloud Desktop Product Documentation](https://support.huaweicloud.com/workspace/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Cloud Desktop Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/desktop/basic)
