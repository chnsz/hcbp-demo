# Deploy Disaster Recovery Drill

## Application Scenario

Business Recovery Service (BRS) is a disaster recovery service for Elastic Cloud Server (ECS) and Elastic Volume Service (EVS). Through host-level replication, data redundancy, and cache acceleration technologies, it provides users with high-level data reliability and business continuity, known as Business Recovery Service.

Business Recovery Service helps protect business applications by replicating Elastic Cloud Server data and configuration information to disaster recovery sites, allowing business applications to start and run normally on disaster recovery site cloud servers during production site cloud server downtime, thereby improving business continuity.

Disaster recovery drills are a very important part of disaster recovery solutions. Through regular disaster recovery drills, you can verify the feasibility of disaster recovery solutions and ensure rapid business recovery when actual failures occur. This best practice will introduce how to use Terraform to automatically deploy a complete BRS disaster recovery drill environment, including protection groups, protected instances, and disaster recovery drills.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS Flavors Query Data Source (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS Images Query Data Source (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)
- [SDRS Domain Query Data Source (data.huaweicloud_sdrs_domain)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/sdrs_domain)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS Instance Resource (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [SDRS Protection Group Resource (huaweicloud_sdrs_protection_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sdrs_protection_group)
- [SDRS Protected Instance Resource (huaweicloud_sdrs_protected_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sdrs_protected_instance)
- [SDRS Disaster Recovery Drill Resource (huaweicloud_sdrs_drill)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sdrs_drill)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── data.huaweicloud_images_images
            └── huaweicloud_compute_instance

data.huaweicloud_sdrs_domain
    └── huaweicloud_sdrs_protection_group
        └── huaweicloud_sdrs_protected_instance
            └── huaweicloud_sdrs_drill

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    │   └── huaweicloud_compute_instance
    └── huaweicloud_sdrs_protection_group

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../docs/introductions/prepare_before_deploy.md).

### 2. Query Availability Zones Required for ECS Instance Resource Creation Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create ECS instances (including for filtering flavors):

```hcl
variable "availability_zone" {
  description = "Availability zone information for ECS instance"
  type        = string
  default     = ""
}

# Get all availability zone information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating ECS instance resources
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- No special parameters, gets all availability zone information in the current region

### 3. Query ECS Instance Flavor Information Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create ECS instances:

```hcl
variable "instance_flavor_id" {
  description = "ECS instance flavor ID"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "ECS instance flavor performance type"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "ECS instance flavor CPU core count"
  type        = number
  default     = 2
}

variable "instance_flavor_memory_size" {
  description = "ECS instance flavor memory size"
  type        = number
  default     = 4
}

# Get all ECS flavor information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating ECS instances
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**Parameter Description**:
- **count**: Data source creation count, used to control whether to execute ECS flavor list query data source, only creates data source when `var.instance_flavor_id` is empty (i.e., execute ECS flavor list query)
- **availability_zone**: Availability zone where ECS instance is located, assigned based on the return result of availability zone list query data source
- **performance_type**: Performance type of ECS instance flavor, assigned by referencing input variable instance_flavor_performance_type
- **cpu_core_count**: CPU core count of ECS instance flavor, assigned by referencing input variable instance_flavor_cpu_core_count
- **memory_size**: Memory size of ECS instance flavor, assigned by referencing input variable instance_flavor_memory_size

### 4. Query ECS Instance Image Information Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create ECS instances:

```hcl
variable "instance_image_id" {
  description = "ECS instance image ID"
  type        = string
  default     = ""
}

variable "instance_image_os_type" {
  description = "ECS instance image operating system type"
  type        = string
  default     = "Ubuntu"
}

variable "instance_image_visibility" {
  description = "ECS instance image visibility"
  type        = string
  default     = "public"
}

# Get all image information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating ECS instances
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].ids[0], "") : var.instance_flavor_id
  os         = var.instance_image_os_type
  visibility = var.instance_image_visibility
}
```

**Parameter Description**:
- **count**: Data source creation count, used to control whether to execute image list query data source, only creates data source when `var.instance_image_id` is empty (i.e., execute image list query)
- **flavor_id**: ECS instance flavor ID, assigned based on the return result of ECS flavor list query data source
- **os**: Operating system type of ECS instance image, assigned by referencing input variable instance_image_os_type
- **visibility**: Visibility of ECS instance image, assigned by referencing input variable instance_image_visibility

### 5. Query SDRS Domain Information Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create SDRS protection groups:

```hcl
variable "sdrs_domain_name" {
  description = "SDRS domain name"
  type        = string
  default     = null
}

# Get all SDRS domain information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating SDRS protection groups
data "huaweicloud_sdrs_domain" "test" {
  name = var.sdrs_domain_name
}
```

**Parameter Description**:
- **name**: SDRS domain name, assigned by referencing input variable sdrs_domain_name

### 6. Create VPC Network

Add the following script to the TF file (such as main.tf) to inform Terraform to create VPC network resources:

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

# Create VPC resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_vpc" "test" {
  count = 2

  name = count.index == 0 ? var.vpc_name : "${var.vpc_name}-drill"
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **count**: Resource creation count, used to create production environment VPC and disaster recovery drill environment VPC
- **name**: VPC name, production environment uses var.vpc_name, disaster recovery drill environment uses "${var.vpc_name}-drill"
- **cidr**: VPC CIDR block, assigned by referencing input variable vpc_cidr

### 7. Create VPC Subnet

Add the following script to the TF file (such as main.tf) to inform Terraform to create VPC subnet resources:

```hcl
variable "subnet_name" {
  description = "Subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "Subnet CIDR block"
  type        = string
  default     = "192.168.0.0/24"
}

variable "subnet_gateway_ip" {
  description = "Subnet gateway IP"
  type        = string
  default     = ""
}

# Create VPC subnet resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_vpc_subnet" "test" {
  count = 2

  vpc_id            = huaweicloud_vpc.test[count.index].id
  name              = count.index == 0 ? var.subnet_name : "${var.subnet_name}-drill"
  cidr              = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test[count.index].cidr, 8, 0) : var.subnet_cidr
  gateway_ip        = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test[count.index].cidr, 8, 0), 1) : var.subnet_gateway_ip
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **count**: Resource creation count, used to create production environment subnet and disaster recovery drill environment subnet
- **vpc_id**: VPC ID to which the subnet belongs, assigned based on the return result of VPC resource
- **name**: Subnet name, production environment uses var.subnet_name, disaster recovery drill environment uses "${var.subnet_name}-drill"
- **cidr**: Subnet CIDR block, assigned by referencing input variable subnet_cidr, automatically calculated if empty
- **gateway_ip**: Subnet gateway IP, assigned by referencing input variable subnet_gateway_ip, automatically calculated if empty
- **availability_zone**: Availability zone where the subnet is located, assigned based on the return result of availability zone list query data source

### 8. Create Security Group

Add the following script to the TF file (such as main.tf) to inform Terraform to create security group resources:

```hcl
variable "security_group_name" {
  description = "Security group name"
  type        = string
}

# Create security group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to true to delete default security group rules

### 9. Create ECS Instance

Add the following script to the TF file (such as main.tf) to inform Terraform to create ECS instance resources:

```hcl
variable "ecs_instance_name" {
  description = "ECS instance name"
  type        = string
}

# Create ECS instance resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  availability_zone  = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, "") : var.instance_flavor_id
  image_id           = var.instance_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, "") : var.instance_image_id
  security_group_ids = [huaweicloud_networking_secgroup.test.id]

  network {
    uuid = huaweicloud_vpc_subnet.test[0].id
  }
}
```

**Parameter Description**:
- **name**: ECS instance name, assigned by referencing input variable ecs_instance_name
- **availability_zone**: Availability zone where ECS instance is located, assigned based on the return result of availability zone list query data source
- **flavor_id**: ID of the flavor used by ECS instance, assigned based on the return result of ECS flavor list query data source
- **image_id**: ID of the image used by ECS instance, assigned based on the return result of IMS image list query data source
- **security_group_ids**: List of security group IDs associated with ECS instance, assigned based on the return result of security group resource
- **network**: Network configuration of ECS instance, associated with production environment subnet

### 10. Create SDRS Protection Group

Add the following script to the TF file (such as main.tf) to inform Terraform to create SDRS protection group resources:

```hcl
variable "protection_group_name" {
  description = "Protection group name"
  type        = string
}

variable "source_availability_zone" {
  description = "Protection group production site availability zone"
  type        = string
  default     = ""

  validation {
    condition = (
      (var.source_availability_zone == "" && var.target_availability_zone == "") ||
      (var.source_availability_zone != "" && var.target_availability_zone != "")
    )
    error_message = "Both `source_availability_zone` and `target_availability_zone` must be set, or both must be empty"
  }
}

variable "target_availability_zone" {
  description = "Protection group disaster recovery site availability zone"
  type        = string
  default     = ""
}

# Create SDRS protection group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_sdrs_protection_group" "test" {
  name                     = var.protection_group_name
  source_availability_zone = var.source_availability_zone != "" ? var.source_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  target_availability_zone = var.target_availability_zone != "" ? var.target_availability_zone : try(data.huaweicloud_availability_zones.test.names[1], null)
  domain_id                = data.huaweicloud_sdrs_domain.test.id
  source_vpc_id            = huaweicloud_vpc.test[0].id
}
```

**Parameter Description**:
- **name**: Protection group name, assigned by referencing input variable protection_group_name
- **source_availability_zone**: Protection group production site availability zone, assigned by referencing input variable source_availability_zone, uses the first availability zone if empty
- **target_availability_zone**: Protection group disaster recovery site availability zone, assigned by referencing input variable target_availability_zone, uses the second availability zone if empty
- **domain_id**: SDRS domain ID, assigned based on the return result of SDRS domain query data source
- **source_vpc_id**: Production site VPC ID, assigned based on the return result of production environment VPC resource

### 11. Create SDRS Protected Instance

Add the following script to the TF file (such as main.tf) to inform Terraform to create SDRS protected instance resources:

```hcl
variable "protected_instance_name" {
  description = "Protected instance name"
  type        = string
}

# Create SDRS protected instance resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_sdrs_protected_instance" "test" {
  name                 = var.protected_instance_name
  group_id             = huaweicloud_sdrs_protection_group.test.id
  server_id            = huaweicloud_compute_instance.test.id
  delete_target_server = true
  delete_target_eip    = true
}
```

**Parameter Description**:
- **name**: Protected instance name, assigned by referencing input variable protected_instance_name
- **group_id**: Protection group ID, assigned based on the return result of SDRS protection group resource
- **server_id**: ECS instance ID, assigned based on the return result of ECS instance resource
- **delete_target_server**: Whether to delete target server, set to true to delete target server
- **delete_target_eip**: Whether to delete target EIP, set to true to delete target EIP

### 12. Create SDRS Disaster Recovery Drill

Add the following script to the TF file (such as main.tf) to inform Terraform to create SDRS disaster recovery drill resources:

```hcl
variable "drill_name" {
  description = "Disaster recovery drill name"
  type        = string
}

# Create SDRS disaster recovery drill resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_sdrs_drill" "test" {
  name         = var.drill_name
  group_id     = huaweicloud_sdrs_protection_group.test.id
  drill_vpc_id = huaweicloud_vpc.test[1].id

  depends_on = [
    huaweicloud_sdrs_protected_instance.test,
    huaweicloud_vpc_subnet.test[1],
  ]
}
```

**Parameter Description**:
- **name**: Disaster recovery drill name, assigned by referencing input variable drill_name
- **group_id**: Protection group ID, assigned based on the return result of SDRS protection group resource
- **drill_vpc_id**: Disaster recovery drill VPC ID, assigned based on the return result of disaster recovery drill environment VPC resource
- **depends_on**: Explicit dependency relationship, ensures protected instance and disaster recovery drill subnet are created before creating disaster recovery drill

### 13. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Basic resource information
vpc_name                = "tf_test_sdrs_protection_instance_vpc"
subnet_name             = "tf_test_sdrs_protection_instance_subnet"
security_group_name     = "tf_test_sdrs_protection_instance_secgroup"
ecs_instance_name       = "tf_test_sdrs_protection_instance_ecs_instance"
protection_group_name   = "tf_test_sdrs_protection_instance_group"
protected_instance_name = "tf_test_sdrs_protected_instance"
drill_name              = "tf_test_sdrs_drill"

# Network configuration
vpc_cidr     = "192.168.0.0/16"
subnet_cidr  = "192.168.0.0/24"

# Instance configuration
instance_flavor_performance_type = "normal"
instance_flavor_cpu_core_count   = 2
instance_flavor_memory_size      = 4
instance_image_os_type           = "Ubuntu"
instance_image_visibility        = "public"

# Disaster recovery configuration
source_availability_zone = ""
target_availability_zone = ""
```

**Usage Method**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content in this `tfvars` file when executing terraform commands, other names need to add `.auto` definition before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 14. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating BRS disaster recovery drill environment
4. Run `terraform show` to view the created BRS disaster recovery drill environment

## Reference Information

- [Huawei Cloud BRS Product Documentation](https://support.huaweicloud.com/sdrs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [BRS Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sdrs)
