# Deploy Cloud Desktop

## Application Scenario

Cloud Desktop (Workspace) is a desktop virtualization service provided by Huawei Cloud, offering enterprises a secure and convenient cloud office environment. With Cloud Desktop, enterprises can achieve centralized data storage and lightweight terminals, and users can securely access cloud office desktops from anywhere through various terminal devices.

This best practice will introduce how to use Terraform to automatically deploy a cloud desktop, including the creation of VPC, subnet, security group, Workspace service, desktop user, and cloud desktop instance, helping you quickly build a cloud office environment.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Workspace Flavors (data.huaweicloud_workspace_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_flavors)
- [Images (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)
- [Workspace Service (data.huaweicloud_workspace_service)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_service)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Workspace Service (huaweicloud_workspace_service)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_service)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Workspace User (huaweicloud_workspace_user)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_user)
- [Workspace Desktop (huaweicloud_workspace_desktop)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_desktop)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_workspace_flavors

data.huaweicloud_images_images

data.huaweicloud_workspace_service
    ├── huaweicloud_vpc
    │   └── huaweicloud_vpc_subnet
    ├── huaweicloud_workspace_service
    └── huaweicloud_networking_secgroup
        └── huaweicloud_networking_secgroup_rule

huaweicloud_workspace_user
    └── huaweicloud_workspace_desktop
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md) article.

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zone to which the cloud desktop flavor and network belong:

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone to which the cloud desktop flavor and network belong"
  type        = string
  default     = ""
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable availability_zone is empty, used to automatically obtain the availability zone list in the current region

### 3. Query Workspace Flavors

Add the following script in the TF file (such as main.tf) to query the cloud desktop flavors that meet the conditions:

```hcl
# Query the Workspace flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

data "huaweicloud_workspace_flavors" "test" {
  count = var.desktop_flavor_id == "" ? 1 : 0

  os_type           = var.desktop_flavor_os_type
  vcpus             = var.desktop_flavor_cpu_core_number
  memory            = var.desktop_flavor_memory_size
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable desktop_flavor_id is empty, used to automatically query the cloud desktop flavor
- **os_type**: Assigned by referencing the input variable desktop_flavor_os_type, indicating the OS type of the cloud desktop flavor
- **vcpus**: Assigned by referencing the input variable desktop_flavor_cpu_core_number, indicating the number of CPU cores of the cloud desktop flavor
- **memory**: Assigned by referencing the input variable desktop_flavor_memory_size, indicating the memory size of the cloud desktop flavor
- **availability_zone**: Assigned by referencing the input variable availability_zone, defaulting to the first queried availability zone when not specified

### 4. Query Workspace Images

Add the following script in the TF file (such as main.tf) to query the cloud desktop images:

```hcl
# Query the Workspace images in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

data "huaweicloud_images_images" "test" {
  count = var.desktop_image_id == "" ? 1 : 0

  name_regex = "WORKSPACE"
  os         = var.desktop_image_os_type
  visibility = var.desktop_image_visibility
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable desktop_image_id is empty, used to automatically query the cloud desktop image
- **name_regex**: The regular expression of the image name, used to filter cloud desktop images
- **os**: Assigned by referencing the input variable desktop_image_os_type, indicating the OS type of the image
- **visibility**: Assigned by referencing the input variable desktop_image_visibility, indicating the visibility of the image

### 5. Query Workspace Service Status

Add the following script in the TF file (such as main.tf) to query the activation status of the Workspace service in the current region:

```hcl
# Query the Workspace service status in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_workspace_service" "test" {}
```

**Parameter Description**:
- This data source requires no additional parameters and is used to obtain the status, VPC ID, network IDs, and security group information of the Workspace service in the current region

### 6. Create a VPC

Add the following script in the TF file (such as main.tf) to create a VPC:

```hcl
# Create a VPC in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **count**: The resource is created when the Workspace service is not activated
- **name**: Assigned by referencing the input variable vpc_name, indicating the VPC name
- **cidr**: Assigned by referencing the input variable vpc_cidr, indicating the CIDR block of the VPC

### 7. Create a VPC Subnet

Add the following script in the TF file (such as main.tf) to create a VPC subnet:

```hcl
# Create a VPC subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

resource "huaweicloud_vpc_subnet" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  vpc_id     = try(huaweicloud_vpc.test[0].id, null)
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(try(huaweicloud_vpc.test[0].cidr, "192.168.0.0/16"), 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(try(huaweicloud_vpc.test[0].cidr, "192.168.0.0/16"), 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **count**: The resource is created when the Workspace service is not activated
- **vpc_id**: Assigned by referencing the VPC ID created in the previous step
- **name**: Assigned by referencing the input variable subnet_name, indicating the subnet name
- **cidr**: Assigned by referencing the input variable subnet_cidr, automatically dividing the subnet CIDR based on the VPC CIDR when not specified
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip, automatically calculating the gateway IP based on the subnet CIDR when not specified

### 8. Activate the Workspace Service

Add the following script in the TF file (such as main.tf) to activate the Workspace service:

```hcl
# Activate the Workspace service in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **count**: The resource is created when the Workspace service is not activated
- **access_mode**: The access mode of the Workspace service, set to `INTERNET` here to indicate access through the internet
- **vpc_id**: Assigned by referencing the VPC ID created in the previous step
- **network_ids**: Assigned by referencing the subnet ID created in the previous step, indicating the network used by the Workspace service

### 9. Create a Security Group

Add the following script in the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **count**: The resource is created when the Workspace service is not activated
- **name**: Assigned by referencing the input variable security_group_name, indicating the security group name
- **delete_default_rules**: Set to `true` to delete the default rules of the security group

### 10. Create a Security Group Rule

Add the following script in the TF file (such as main.tf) to create a security group rule:

```hcl
# Create a security group rule in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **count**: The resource is created when the Workspace service is not activated
- **security_group_id**: Assigned by referencing the security group ID created in the previous step
- **direction**: The direction of the rule, set to `egress` here to indicate the outbound direction
- **ethertype**: The IP protocol type, set to `IPv4` here
- **remote_ip_prefix**: The remote IP address range, set to `0.0.0.0/0` here to allow all addresses
- **priority**: The priority of the rule; a smaller value indicates a higher priority

### 11. Create a Workspace User

Add the following script in the TF file (such as main.tf) to create a Workspace user:

```hcl
# Create a Workspace user in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "desktop_user_name" {
  description = "The user name that the cloud desktop used"
  type        = string
}

variable "desktop_user_email" {
  description = "The email address that the user used"
  type        = string
}

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
- **depends_on**: Explicitly depends on the Workspace service to ensure the user is created after the service is activated
- **name**: Assigned by referencing the input variable desktop_user_name, indicating the user name
- **email**: Assigned by referencing the input variable desktop_user_email, indicating the user email
- **account_expires**: The account expiration time, set to `0` to indicate never expires
- **password_never_expires**: Whether the password never expires, set to `false` here
- **enable_change_password**: Whether to allow password changes, set to `true` here
- **next_login_change_password**: Whether to change the password on the next login, set to `true` here
- **disabled**: Whether to disable the user, set to `false` here

### 12. Create a Cloud Desktop

Add the following script in the TF file (such as main.tf) to create a cloud desktop:

```hcl
# Create a cloud desktop in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  type        = list(object({
    type = string
    size = number
  }))
  default     = [
    {
      type = "SSD",
      size = 100,
    },
  ]
}

resource "huaweicloud_workspace_desktop" "test" {
  depends_on = [huaweicloud_workspace_user.test]

  flavor_id         = var.desktop_flavor_id == "" ? try([for o in data.huaweicloud_workspace_flavors.test[0].flavors : o.id if !strcontains(lower(o.description), "flexus")][0], null) : var.desktop_flavor_id
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
- **depends_on**: Explicitly depends on the Workspace user to ensure the desktop is created after the user is created
- **flavor_id**: Assigned by referencing the input variable desktop_flavor_id; when not specified, automatically filters non-Flexus flavors from the flavor list
- **image_type**: Assigned by referencing the input variable desktop_image_visibility, indicating the image type
- **image_id**: Assigned by referencing the input variable desktop_image_id; when not specified, automatically uses the first queried image
- **availability_zone**: Assigned by referencing the input variable availability_zone; when not specified, defaults to the first queried availability zone
- **vpc_id**: Uses the VPC associated with the Workspace service when the service is activated, otherwise uses the created VPC
- **security_groups**: The list of security groups used by the cloud desktop, including the desktop security group, infrastructure security group, and the created security group
- **nic**: The NIC configuration of the cloud desktop, referencing the network IDs of the Workspace service or the created subnet ID through a dynamic block
- **name**: Assigned by referencing the input variable cloud_desktop_name, indicating the cloud desktop name
- **user_name**: Assigned by referencing the name of the Workspace user
- **user_email**: Assigned by referencing the email of the Workspace user
- **user_group**: Assigned by referencing the input variable desktop_user_group_name, indicating the user group the user belongs to
- **root_volume**: The system disk configuration; **type** is assigned by referencing the input variable desktop_root_volume_type, and **size** is assigned by referencing the input variable desktop_root_volume_size
- **data_volume**: The data disk configuration, assigned by referencing the input variable desktop_data_volumes through a dynamic block
- **lifecycle**: Ignores changes to flavor_id, image_id, and availability_zone to avoid resource recreation caused by changes in automatic query results

### 13. Preset Input Parameters Required for Resource Deployment (Optional)

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
vpc_cidr            = "192.168.0.0/16"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
desktop_user_name   = "tf_test_user"
desktop_user_email  = "test@example.com"
cloud_desktop_name  = "tf-test-desktop"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 14. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the cloud desktop
4. Run `terraform show` to view the created cloud desktop

## Reference Information

- [Huawei Cloud Workspace Product Documentation](https://support.huaweicloud.com/workspace/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Workspace Cloud Desktop](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/desktop/basic)
