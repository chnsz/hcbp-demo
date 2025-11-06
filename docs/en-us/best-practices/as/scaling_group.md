# Deploy Scaling Group

## Application Scenario

Huawei Cloud Auto Scaling service is a service that automatically adjusts computing resources, capable of automatically adjusting the number of elastic computing instances based on business needs and policies. Scaling group is the core resource of Auto Scaling service, used to manage a group of elastic computing instances with the same configuration and rules. By creating a scaling group, you can automatically manage the increase and decrease of instance numbers, realize elastic expansion and contraction of resources, and meet the needs of business load changes. This best practice will introduce how to use Terraform to automatically deploy a scaling group, including querying availability zones, instance flavors, and images, as well as creating security groups, key pairs, AS configuration, VPC network, and scaling group.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Compute Flavors Query Data Source (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [Images Query Data Source (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Key Pair Resource (huaweicloud_kps_keypair)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [AS Configuration Resource (huaweicloud_as_configuration)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_configuration)
- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [AS Group Resource (huaweicloud_as_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_group)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── data.huaweicloud_images_images
            └── huaweicloud_as_configuration

huaweicloud_networking_secgroup
    └── huaweicloud_as_configuration
        └── huaweicloud_as_group

huaweicloud_kps_keypair
    └── huaweicloud_as_configuration
        └── huaweicloud_as_group

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_as_group
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones Required for Scaling Group Resource Creation Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create scaling group related resources:

```hcl
variable "availability_zone" {
  description = "The availability zone to which the scaling configuration belongs"
  type        = string
  default     = ""
}

# Get all availability zone information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating scaling group related resources
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: The number of data source creations, used to control whether to execute the availability zone list query, only when `var.availability_zone` is empty, the data source is created (i.e., execute the availability zone list query)

### 3. Query Instance Flavors Required for Scaling Group Resource Creation Through Data Source

Add the following script to the TF file to inform Terraform to query instance flavors that meet the conditions:

```hcl
variable "configuration_flavor_id" {
  description = "The flavor ID of the scaling configuration"
  type        = string
  default     = ""
}

variable "configuration_flavor_performance_type" {
  description = "The performance type of the scaling configuration"
  type        = string
  default     = "normal"
}

variable "configuration_flavor_cpu_core_count" {
  description = "The CPU core count of the scaling configuration"
  type        = number
  default     = 2
}

variable "configuration_flavor_memory_size" {
  description = "The memory size of the scaling configuration"
  type        = number
  default     = 4
}

# Get all instance flavor information that meets specific conditions in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating scaling group related resources
data "huaweicloud_compute_flavors" "test" {
  count = var.configuration_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  performance_type  = var.configuration_flavor_performance_type
  cpu_core_count    = var.configuration_flavor_cpu_core_count
  memory_size       = var.configuration_flavor_memory_size
}
```

**Parameter Description**:
- **count**: The number of data source creations, used to control whether to execute the instance flavor list query, only when `var.configuration_flavor_id` is empty, the data source is created (i.e., execute the instance flavor list query)
- **availability_zone**: The availability zone where the instance flavor is located, if the availability zone is specified, use that value, otherwise use the first availability zone from the availability zone list query data source
- **performance_type**: Performance type, assigned through input variable `configuration_flavor_performance_type`, default is "normal" for standard type
- **cpu_core_count**: CPU core count, assigned through input variable `configuration_flavor_cpu_core_count`
- **memory_size**: Memory size (GB), assigned through input variable `configuration_flavor_memory_size`

### 4. Query Images Required for Scaling Group Resource Creation Through Data Source

Add the following script to the TF file to inform Terraform to query images that meet the conditions:

```hcl
variable "configuration_image_id" {
  description = "The image ID of the scaling configuration"
  type        = string
  default     = ""
}

variable "configuration_image_visibility" {
  description = "The visibility of the image"
  type        = string
  default     = "public"
}

variable "configuration_image_os" {
  description = "The OS of the image"
  type        = string
  default     = "Ubuntu"
}

# Get all image information that meets specific conditions in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating scaling group related resources
data "huaweicloud_images_images" "test" {
  count = var.configuration_image_id == "" ? 1 : 0

  flavor_id  = var.configuration_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null) : var.configuration_flavor_id
  visibility = var.configuration_image_visibility
  os         = var.configuration_image_os
}
```

**Parameter Description**:
- **count**: The number of data source creations, used to control whether to execute the image list query, only when `var.configuration_image_id` is empty, the data source is created (i.e., execute the image list query)
- **flavor_id**: The flavor ID supported by the image, if the flavor ID is specified, use that value, otherwise use the first flavor ID from the instance flavor list query data source
- **visibility**: Image visibility, assigned through input variable `configuration_image_visibility`, default is "public" for public images
- **os**: Operating system type, assigned through input variable `configuration_image_os`, default is "Ubuntu"

### 5. Create Security Group Resource

Add the following script to the TF file to inform Terraform to create security group resources:

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# Create security group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for providing security protection for scaling group
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: The name of the security group, assigned by referencing input variable `security_group_name`
- **delete_default_rules**: Whether to delete default rules, set to true to delete default rules

### 6. Create Key Pair Resource

Add the following script to the TF file to inform Terraform to create key pair resources:

```hcl
variable "keypair_name" {
  description = "The name of the key pair"
  type        = string
}

variable "keypair_public_key" {
  description = "The public key for SSH access"
  type        = string
  default     = ""
}

# Create key pair resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for accessing AS instances
resource "huaweicloud_kps_keypair" "test" {
  name       = var.keypair_name
  public_key = var.keypair_public_key != "" ? var.keypair_public_key : null
}
```

**Parameter Description**:
- **name**: The name of the key pair, assigned by referencing input variable `keypair_name`
- **public_key**: The public key of the key pair, if the public key is specified, use that value, otherwise set to null (the system will automatically generate a key pair)

### 7. Create AS Configuration Resource

Add the following script to the TF file to inform Terraform to create AS configuration resources:

```hcl
variable "configuration_name" {
  description = "The name of the scaling configuration"
  type        = string
}

variable "configuration_metadata" {
  description = "The metadata for the scaling configuration instances"
  type        = map(string)
}

variable "configuration_user_data" {
  description = "The user data script for scaling configuration instances initialization"
  type        = string
}

variable "configuration_disks" {
  description = "The disk configurations for the scaling configuration instances"
  type = list(object({
    size        = number
    volume_type = string
    disk_type   = string
  }))

  nullable = false
}

variable "configuration_public_eip_settings" {
  description = "The public IP settings for the scaling configuration instances"
  type = list(object({
    ip_type = string
    bandwidth = object({
      size          = number
      share_type    = string
      charging_mode = string
    })
  }))

  nullable = false
  default  = []
}

# Create AS configuration resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for defining templates for AS instances
resource "huaweicloud_as_configuration" "test" {
  scaling_configuration_name = var.configuration_name

  instance_config {
    image              = var.configuration_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, null) : var.configuration_image_id
    flavor             = var.configuration_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null) : var.configuration_flavor_id
    key_name           = huaweicloud_kps_keypair.test.id
    security_group_ids = [huaweicloud_networking_secgroup.test.id]

    metadata  = var.configuration_metadata
    user_data = var.configuration_user_data

    dynamic "disk" {
      for_each = var.configuration_disks

      content {
        size        = disk.value["size"]
        volume_type = disk.value["volume_type"]
        disk_type   = disk.value["disk_type"]
      }
    }

    dynamic "public_ip" {
      for_each = var.configuration_public_eip_settings

      content {
        eip {
          ip_type = public_ip.value.ip_type

          bandwidth {
            size          = public_ip.value.bandwidth.size
            share_type    = public_ip.value.bandwidth.share_type
            charging_mode = public_ip.value.bandwidth.charging_mode
          }
        }
      }
    }
  }
}
```

**Parameter Description**:
- **scaling_configuration_name**: The name of the AS configuration, assigned by referencing input variable `configuration_name`
- **instance_config**: Instance configuration block, defines the configuration for AS instances
  - **image**: Instance image ID, if the image ID is specified, use that value, otherwise use the first image ID from the image list query data source
  - **flavor**: Instance flavor ID, if the flavor ID is specified, use that value, otherwise use the first flavor ID from the instance flavor list query data source
  - **key_name**: Key pair ID, assigned by referencing the ID of the key pair resource (huaweicloud_kps_keypair.test)
  - **security_group_ids**: Security group ID list, assigned by referencing the ID of the security group resource (huaweicloud_networking_secgroup.test)
  - **metadata**: Instance metadata, assigned through input variable `configuration_metadata`, used to inject custom data when instances start
  - **user_data**: User data script, assigned through input variable `configuration_user_data`, used to execute initialization scripts when instances start
  - **disk**: Disk configuration block, creates multiple disk configurations through dynamic block (dynamic block) based on input variable `configuration_disks`
    - **size**: Disk size (GB), assigned through `size` in input variable `configuration_disks`
    - **volume_type**: Volume type, assigned through `volume_type` in input variable `configuration_disks`
    - **disk_type**: Disk type, assigned through `disk_type` in input variable `configuration_disks`
  - **public_ip**: Public IP configuration block, creates public IP configuration through dynamic block (dynamic block) based on input variable `configuration_public_eip_settings`
    - **eip**: Elastic IP configuration block
      - **ip_type**: IP type, assigned through `ip_type` in input variable `configuration_public_eip_settings`
      - **bandwidth**: Bandwidth configuration block
        - **size**: Bandwidth size (Mbps), assigned through `bandwidth.size` in input variable `configuration_public_eip_settings`
        - **share_type**: Share type, assigned through `bandwidth.share_type` in input variable `configuration_public_eip_settings`
        - **charging_mode**: Charging mode, assigned through `bandwidth.charging_mode` in input variable `configuration_public_eip_settings`

> Note: The disk configuration must include at least one system disk (disk_type is "SYS"), and other disks are data disks. Public IP configuration is optional, if public IP is not needed, you can not configure `configuration_public_eip_settings` or set it to an empty list.

### 8. Create VPC Resource (Optional)

Add the following script to the TF file to inform Terraform to create VPC resources (if VPC ID is not specified):

```hcl
variable "scaling_group_vpc_id" {
  description = "The ID of the VPC"
  type        = string
  default     = ""
}

variable "scaling_group_vpc_name" {
  description = "The name of the VPC"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.scaling_group_vpc_id == "" && var.scaling_group_vpc_name != ""
    error_message = "When 'scaling_group_vpc_id' is empty, 'scaling_group_vpc_name' is not allowed to be empty"
  }
}

variable "scaling_group_vpc_cidr" {
  description = "The CIDR of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

# Create VPC resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for providing network environment for scaling group
resource "huaweicloud_vpc" "test" {
  count = var.scaling_group_vpc_id == "" && var.scaling_group_subnet_id == "" ? 1 : 0

  name = var.scaling_group_vpc_name
  cidr = var.scaling_group_vpc_cidr
}
```

**Parameter Description**:
- **count**: The number of resource creations, used to control whether to create VPC resource, only when both `var.scaling_group_vpc_id` and `var.scaling_group_subnet_id` are empty, the VPC resource is created
- **name**: The name of the VPC, assigned by referencing input variable `scaling_group_vpc_name`
- **cidr**: The CIDR block of the VPC, assigned by referencing input variable `scaling_group_vpc_cidr`, default is "192.168.0.0/16"

### 9. Create VPC Subnet Resource (Optional)

Add the following script to the TF file to inform Terraform to create VPC subnet resources (if subnet ID is not specified):

```hcl
variable "scaling_group_subnet_id" {
  description = "The ID of the subnet"
  type        = string
  default     = ""
}

variable "scaling_group_subnet_name" {
  description = "The name of the subnet"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.scaling_group_subnet_id == "" && var.scaling_group_subnet_name != ""
    error_message = "When 'scaling_group_subnet_id' is empty, 'scaling_group_subnet_name' is not allowed to be empty"
  }
}

variable "scaling_group_subnet_cidr" {
  description = "The CIDR of the subnet"
  type        = string
  default     = ""
}

variable "scaling_group_subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
}

# Create VPC subnet resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for providing network environment for scaling group
resource "huaweicloud_vpc_subnet" "test" {
  count = var.scaling_group_subnet_id == "" ? 1 : 0

  vpc_id     = var.scaling_group_vpc_id == "" ? huaweicloud_vpc.test[0].id : var.scaling_group_vpc_id
  name       = var.scaling_group_subnet_name
  cidr       = var.scaling_group_subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test[0].cidr, 8, 0) : var.scaling_group_subnet_cidr
  gateway_ip = var.scaling_group_subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test[0].cidr, 8, 0), 1) : var.scaling_group_subnet_gateway_ip
}
```

**Parameter Description**:
- **count**: The number of resource creations, used to control whether to create VPC subnet resource, only when `var.scaling_group_subnet_id` is empty, the VPC subnet resource is created
- **vpc_id**: The VPC ID to which the subnet belongs, if the VPC ID is specified, use that value, otherwise assign by referencing the ID of the VPC resource (huaweicloud_vpc.test[0])
- **name**: The name of the subnet, assigned by referencing input variable `scaling_group_subnet_name`
- **cidr**: The CIDR block of the subnet, if the subnet CIDR is specified, use that value, otherwise automatically calculate based on the VPC's CIDR block using the `cidrsubnet` function
- **gateway_ip**: The gateway IP of the subnet, if the gateway IP is specified, use that value, otherwise automatically calculate based on the subnet CIDR using the `cidrhost` function

### 10. Create AS Group Resource

Add the following script to the TF file to inform Terraform to create AS group resources:

```hcl
variable "scaling_group_name" {
  description = "The name of the scaling group"
  type        = string
}

variable "scaling_group_desire_instance_number" {
  description = "The desired number of instances"
  type        = number
  default     = 2
}

variable "scaling_group_min_instance_number" {
  description = "The minimum number of instances"
  type        = number
  default     = 0
}

variable "scaling_group_max_instance_number" {
  description = "The maximum number of instances"
  type        = number
  default     = 10
}

variable "is_delete_scaling_group_publicip" {
  description = "Whether to delete the public IP address of the scaling instances when the scaling group is deleted"
  type        = bool
  default     = true
}

variable "is_delete_scaling_group_instances" {
  description = "Whether to delete the scaling instances when the scaling group is deleted"
  type        = string
  default     = "yes"
}

# Create AS group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for managing AS instances
resource "huaweicloud_as_group" "test" {
  scaling_group_name       = var.scaling_group_name
  scaling_configuration_id = huaweicloud_as_configuration.test.id
  desire_instance_number   = var.scaling_group_desire_instance_number
  min_instance_number      = var.scaling_group_min_instance_number
  max_instance_number      = var.scaling_group_max_instance_number
  vpc_id                   = var.scaling_group_vpc_id == "" ? huaweicloud_vpc.test[0].id : var.scaling_group_vpc_id
  delete_publicip          = var.is_delete_scaling_group_publicip
  delete_instances         = var.is_delete_scaling_group_instances

  networks {
    id = var.scaling_group_subnet_id == "" ? huaweicloud_vpc_subnet.test[0].id : var.scaling_group_subnet_id
  }
}
```

**Parameter Description**:
- **scaling_group_name**: The name of the AS group, assigned by referencing input variable `scaling_group_name`
- **scaling_configuration_id**: AS configuration ID, assigned by referencing the ID of the AS configuration resource (huaweicloud_as_configuration.test)
- **desire_instance_number**: The desired number of AS instances, assigned by referencing input variable `scaling_group_desire_instance_number`, default is 2
- **min_instance_number**: The minimum number of AS instances, assigned by referencing input variable `scaling_group_min_instance_number`, default is 0
- **max_instance_number**: The maximum number of AS instances, assigned by referencing input variable `scaling_group_max_instance_number`, default is 10
- **vpc_id**: VPC ID, if the VPC ID is specified, use that value, otherwise assign by referencing the ID of the VPC resource (huaweicloud_vpc.test[0])
- **delete_publicip**: Whether to delete the public IP address of AS instances when the AS group is deleted, assigned by referencing input variable `is_delete_scaling_group_publicip`, default is true
- **delete_instances**: Whether to delete AS instances when the AS group is deleted, assigned by referencing input variable `is_delete_scaling_group_instances`, value is "yes" or "no", default is "yes"
- **networks**: Network configuration block, defines the subnet used by the AS group
  - **id**: Subnet ID, if the subnet ID is specified, use that value, otherwise assign by referencing the ID of the VPC subnet resource (huaweicloud_vpc_subnet.test[0])

> Note: `min_instance_number`, `desire_instance_number` and `max_instance_number` must satisfy the relationship: `min_instance_number <= desire_instance_number <= max_instance_number`.

### 11. Preset Input Parameters Required for Resource Deployment

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, example content is as follows:

```hcl
security_group_name     = "tf_test_secgroup_demo"
keypair_name            = "tf_test_keypair_demo"
configuration_name      = "tf_test_as_configuration"
configuration_metadata  = {
  some_key = "some_value"
}
configuration_user_data = <<EOT
# !/bin/sh
echo "Hello World! The time is now $(date -R)!" | tee /root/output.txt
EOT

configuration_disks = [
  {
    size        = 40
    volume_type = "SSD"
    disk_type   = "SYS"
  }
]

configuration_public_eip_settings = [
  {
    ip_type   = "5_bgp"
    bandwidth = {
      size          = 10
      share_type    = "PER"
      charging_mode = "traffic"
    }
  }
]

scaling_group_vpc_name    = "tf_test_vpc_demo"
scaling_group_subnet_name = "tf_test_subnet_demo"
scaling_group_name        = "tf_test_scaling_group_demo"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content in the `tfvars` file when executing terraform commands, other naming requires adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="security_group_name=my-secgroup" -var="keypair_name=my-keypair"`
2. Environment variables: `export TF_VAR_security_group_name=my-secgroup`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating scaling group
4. Run `terraform show` to view the created scaling group

## Reference Information

- [Huawei Cloud Auto Scaling Product Documentation](https://support.huaweicloud.com/as/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [AS Scaling Group Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/as/scaling-group)
