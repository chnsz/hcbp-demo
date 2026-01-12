# Deploy Associate ECS Instance

## Application Scenario

Dedicated Host (DEH) is a physical server resource provided by Huawei Cloud, used to meet business scenarios with special requirements for resource exclusivity, security compliance, etc. By deploying ECS instances on dedicated hosts, you can obtain full control of physical servers, achieve physical isolation of resources, and meet compliance requirements. Automating ECS instance deployment on dedicated hosts through Terraform can ensure standardized and consistent resource deployment, improving operational efficiency. This best practice will introduce how to use Terraform to automatically deploy ECS instances on dedicated hosts.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Data Sources

- [Availability Zones Data Source (huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Dedicated Host Types Data Source (huaweicloud_deh_types)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/deh_types)
- [Images Data Source (huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [Dedicated Host Instance Resource (huaweicloud_deh_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/deh_instance)
- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS Instance Resource (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)

## Resource Dependencies

In this best practice, the following dependencies exist between resources:

1. **ECS Instance** depends on **Dedicated Host Instance**, which needs to be associated with the dedicated host through scheduler_hints
2. **ECS Instance** depends on **VPC Subnet** and **Security Group**, which need to be created first
3. **VPC Subnet** depends on **VPC**, which needs to be created first
4. **Dedicated Host Instance** depends on **Availability Zones Data Source** and **Dedicated Host Types Data Source** to obtain availability zone and host type information
5. **ECS Instance** depends on **Images Data Source** to obtain image information

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Data Sources

Add the following script to the TF file (e.g., main.tf) to query availability zone, dedicated host type and image information:

```hcl
# Query availability zone information
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

# Query dedicated host type information
data "huaweicloud_deh_types" "test" {
  count = var.deh_instance_host_type == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}

# Query image information
data "huaweicloud_images_images" "test" {
  count = var.ecs_instance_image_id == "" ? 1 : 0

  flavor_id  = var.ecs_instance_flavor_id != "" ? var.ecs_instance_flavor_id : try(huaweicloud_deh_instance.test.host_properties[0].available_instance_capacities[0].flavor, null)
  visibility = var.ecs_instance_image_visibility
  os         = var.ecs_instance_image_os
}
```

**Parameter Description**:
- **availability_zone**: Availability zone name, queried when availability_zone variable is empty
- **host_type**: Dedicated host type, queried when deh_instance_host_type variable is empty
- **flavor_id**: ECS flavor ID, used to query matching images
- **visibility**: Image visibility, assigned by referencing the input variable ecs_instance_image_visibility, default value is "public"
- **os**: Image operating system, assigned by referencing the input variable ecs_instance_image_os, default value is "Ubuntu"

### 3. Create Dedicated Host Instance Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a dedicated host instance resource:

```hcl
variable "deh_instance_name" {
  description = "The name of the dedicated host instance"
  type        = string
}

variable "availability_zone" {
  description = "The availability zone where the resources will be created"
  type        = string
  default     = ""
}

variable "deh_instance_host_type" {
  description = "The host type of the dedicated host"
  type        = string
  default     = ""
}

variable "deh_instance_auto_placement" {
  description = "Whether to enable auto placement for the dedicated host"
  type        = string
  default     = "on"
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the dedicated host"
  type        = string
  default     = null
}

variable "deh_instance_charging_mode" {
  description = "The charging mode of the dedicated host"
  type        = string
  default     = "prePaid"
}

variable "deh_instance_period_unit" {
  description = "The unit of the billing period of the dedicated host"
  type        = string
  default     = "month"
}

variable "deh_instance_period" {
  description = "The billing period of the dedicated host"
  type        = string
  default     = "1"
}

variable "deh_instance_auto_renew" {
  description = "Whether to enable auto renew for the dedicated host"
  type        = string
  default     = "false"
}

# Create dedicated host instance resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_deh_instance" "test" {
  name                  = var.deh_instance_name
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  host_type             = var.deh_instance_host_type != "" ? var.deh_instance_host_type : try(data.huaweicloud_deh_types.test[0].dedicated_host_types[0].host_type, null)
  auto_placement        = var.deh_instance_auto_placement
  enterprise_project_id = var.enterprise_project_id
  charging_mode         = var.deh_instance_charging_mode
  period_unit           = var.deh_instance_period_unit
  period                = var.deh_instance_period
  auto_renew            = var.deh_instance_auto_renew
}
```

**Parameter Description**:
- **name**: Dedicated host instance name, assigned by referencing the input variable deh_instance_name
- **availability_zone**: Availability zone name, assigned by referencing the input variable availability_zone or availability zones data source
- **host_type**: Dedicated host type, assigned by referencing the input variable deh_instance_host_type or dedicated host types data source
- **auto_placement**: Whether to enable auto placement, assigned by referencing the input variable deh_instance_auto_placement, default value is "on"
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id, optional parameter, default value is null
- **charging_mode**: Charging mode, assigned by referencing the input variable deh_instance_charging_mode, default value is "prePaid" (prepaid)
- **period_unit**: Billing period unit, assigned by referencing the input variable deh_instance_period_unit, default value is "month"
- **period**: Billing period, assigned by referencing the input variable deh_instance_period, default value is "1"
- **auto_renew**: Whether to enable auto renew, assigned by referencing the input variable deh_instance_auto_renew, default value is "false"

### 4. Create Basic Network Resources

Add the following script to the TF file (e.g., main.tf) to create VPC, subnet and security group:

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

variable "subnet_name" {
  description = "The name of the VPC subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the VPC subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the VPC subnet"
  type        = string
  default     = ""
}

variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# Create VPC
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

# Create subnet
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}

# Create security group
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

### 5. Create ECS Instance Resource Associated with Dedicated Host

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an ECS instance resource associated with the dedicated host:

```hcl
variable "ecs_instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "ecs_instance_image_id" {
  description = "The ID of the ECS instance image"
  type        = string
  default     = ""
}

variable "ecs_instance_flavor_id" {
  description = "The ID of the ECS instance flavor"
  type        = string
  default     = ""
}

variable "ecs_instance_image_visibility" {
  description = "The visibility of the ECS instance image"
  type        = string
  default     = "public"
}

variable "ecs_instance_image_os" {
  description = "The OS of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

variable "ecs_instance_admin_pass" {
  description = "The password of the ECS instance administrator"
  type        = string
  sensitive   = true
}

# Create ECS instance resource associated with dedicated host in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  availability_zone  = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  flavor_id          = var.ecs_instance_flavor_id != "" ? var.ecs_instance_flavor_id : try(huaweicloud_deh_instance.test.host_properties[0].available_instance_capacities[0].flavor, null)
  image_id           = var.ecs_instance_image_id != "" ? var.ecs_instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, "")
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  admin_pass         = var.ecs_instance_admin_pass

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  scheduler_hints {
    tenancy = "dedicated"
    deh_id  = huaweicloud_deh_instance.test.id
  }
}
```

**Parameter Description**:
- **name**: ECS instance name, assigned by referencing the input variable ecs_instance_name
- **availability_zone**: Availability zone name, assigned by referencing the input variable availability_zone or availability zones data source
- **flavor_id**: ECS flavor ID, assigned by referencing the input variable ecs_instance_flavor_id or available flavors from the dedicated host instance
- **image_id**: Image ID, assigned by referencing the input variable ecs_instance_image_id or images data source
- **security_group_ids**: Security group ID list, assigned by referencing the security group resource
- **admin_pass**: Administrator password, assigned by referencing the input variable ecs_instance_admin_pass
- **network.uuid**: Network subnet ID, assigned by referencing the subnet resource
- **scheduler_hints.tenancy**: Tenant type, set to "dedicated" to deploy on dedicated host
- **scheduler_hints.deh_id**: Dedicated host ID, assigned by referencing the dedicated host instance resource

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Dedicated Host Instance Configuration
deh_instance_name       = "tf_test_deh_instance"

# VPC and Subnet Configuration
vpc_name                = "tf_test_vpc"
subnet_name             = "tf_test_subnet"

# Security Group Configuration
security_group_name     = "tf_test_security_group"

# ECS Instance Configuration
ecs_instance_name       = "tf_test_ecs_instance"
ecs_instance_admin_pass = "YourPassword@12!"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially `ecs_instance_admin_pass` needs to be set to a password that meets password complexity requirements
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="deh_instance_name=my_deh" -var="ecs_instance_name=my_ecs"`
2. Environment variables: `export TF_VAR_deh_instance_name=my_deh` and `export TF_VAR_ecs_instance_name=my_ecs`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create the dedicated host instance and associated ECS instance:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the dedicated host instance and ECS instance
4. Run `terraform show` to view the details of the created dedicated host instance and ECS instance

> Note: The ECS instance flavor must match the dedicated host type. Currently, only on-demand (postPaid) ECS instances are supported on dedicated hosts. By configuring scheduler_hints, ECS instances can be associated with specified dedicated hosts to achieve physical isolation of resources.

## Reference Information

- [Huawei Cloud DEH Product Documentation](https://support.huaweicloud.com/deh/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Associate ECS Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/deh/associate-ecs-instance)
