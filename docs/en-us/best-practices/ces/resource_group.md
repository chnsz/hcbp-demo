# Deploy Resource Group

## Application Scenario

Cloud Eye Service (CES) resource group is a resource grouping management function provided by the CES service, used to group multiple cloud resources according to business needs. By configuring resource groups, you can organize related cloud resources (such as ECS instances, RDS instances, etc.) together for unified monitoring, alarm and operation management, improving the efficiency and convenience of resource management. Automating CES resource group creation through Terraform can ensure standardized and consistent resource grouping configuration, simplifying operational management. This best practice will introduce how to use Terraform to automatically create CES resource groups.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Data Sources

- [Availability Zones Data Source (huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS Flavors Data Source (huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS Instance Resource (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [CES Resource Group Resource (huaweicloud_ces_resource_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_resource_group)

## Resource Dependencies

In this best practice, the following dependencies exist between resources:

1. **CES Resource Group** depends on cloud resources such as **ECS Instance**, which need to be created first before they can be added to the resource group
2. **ECS Instance** depends on **VPC Subnet** and **Security Group**, which need to be created first
3. **VPC Subnet** depends on **VPC**, which needs to be created first
4. **ECS Instance** depends on **Availability Zones Data Source** and **ECS Flavors Data Source** to obtain availability zone and flavor information

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Data Sources

Add the following script to the TF file (e.g., main.tf) to query availability zone and ECS flavor information:

```hcl
# Query availability zone information
data "huaweicloud_availability_zones" "test" {}

# Query ECS flavor information
data "huaweicloud_compute_flavors" "test" {
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.ecs_flavor_performance_type
  cpu_core_count    = var.ecs_flavor_cpu_core_count
  memory_size       = var.ecs_flavor_memory_size
}
```

**Parameter Description**:
- **availability_zone**: Availability zone name, obtained by referencing the availability zones data source
- **performance_type**: ECS flavor performance type, assigned by referencing the input variable ecs_flavor_performance_type, default value is "normal"
- **cpu_core_count**: ECS flavor CPU core count, assigned by referencing the input variable ecs_flavor_cpu_core_count, default value is 2
- **memory_size**: ECS flavor memory size, assigned by referencing the input variable ecs_flavor_memory_size, default value is 4

### 3. Create Basic Resources

Add the following script to the TF file (e.g., main.tf) to create VPC, subnet, security group and ECS instance:

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
  description = "The name of the subnet"
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

variable "security_group_name" {
  description = "The security group name"
  type        = string
  default     = "sg-dnat-backend"
}

variable "ecs_instance_name" {
  description = "The name of the ECS instance"
  type        = string
  default     = ""
}

variable "ecs_image_id" {
  description = "The ID of the ECS image"
  type        = string
  default     = ""
}

variable "ecs_flavor_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "ecs_flavor_cpu_core_count" {
  description = "The CPU core count of the ECS instance flavor"
  type        = number
  default     = 2
}

variable "ecs_flavor_memory_size" {
  description = "The memory size of the ECS instance flavor"
  type        = number
  default     = 4
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

# Create ECS instance
resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  image_id           = var.ecs_image_id
  flavor_id          = try(data.huaweicloud_compute_flavors.test.ids[0], null)
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }
}
```

### 4. Create CES Resource Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CES resource group resource:

```hcl
variable "resource_group_name" {
  description = "The name of the resource group"
  type        = string
}

variable "resource_group_resources" {
  description = "The resource list of the CES resource group"
  type = list(object({
    namespace  = string
    dimensions = list(object({
      name  = string
      value = string
    }))
  }))
  default = []
}

# Create CES resource group resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_ces_resource_group" "test" {
  name = var.resource_group_name

  dynamic "resources" {
    for_each = var.resource_group_resources

    content {
      namespace = resources.value.namespace
      dimensions {
        name  = resources.value.dimensions[0].name
        value = resources.value.dimensions[0].value
      }
    }
  }
}
```

**Parameter Description**:
- **name**: The resource group name, assigned by referencing the input variable resource_group_name
- **resources**: The resource list, assigned by referencing the input variable resource_group_resources, optional parameter, default value is empty list, each resource contains the following parameters:
  - **namespace**: Service namespace, such as SYS.ECS, SYS.RDS, etc.
  - **dimensions**: Resource dimension list, each dimension contains the following fields:
    - **name**: Dimension name, such as instance_id, rds_instance_id, etc.
    - **value**: Dimension value, such as ECS instance ID, RDS instance ID, etc.

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# VPC and Subnet Configuration
vpc_name    = "tf_test_ces_resource_group"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_ces_resource_group"

# Security Group Configuration
security_group_name = "tf_test_ces_resource_group"

# ECS Instance Configuration
ecs_instance_name = "tf_test_ces_resource_group"
ecs_image_id      = "your_image_id"

# Resource Group Configuration
resource_group_name = "tf_test_ces_resource_group"

# Resource Group Resources Configuration (Example: Add ECS instance to resource group)
resource_group_resources = [
  {
    namespace = "SYS.ECS"
    dimensions = [
      {
        name  = "instance_id"
        value = huaweicloud_compute_instance.test.id
      }
    ]
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially `ecs_image_id` needs to be replaced with the actual image ID
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="resource_group_name=my_group" -var="vpc_name=my_vpc"`
2. Environment variables: `export TF_VAR_resource_group_name=my_group` and `export TF_VAR_vpc_name=my_vpc`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a CES resource group:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the resource group and related resources
4. Run `terraform show` to view the details of the created resource group

> Note: After the resource group is created, multiple cloud resources can be added to the resource group for unified management. Resource groups support resource filtering by namespace and dimensions, allowing flexible organization and management of cloud resources. After the resource group is created, you can further configure it in the CES console, such as creating alarm rules, viewing monitoring data, etc.

## Reference Information

- [Huawei Cloud CES Product Documentation](https://support.huaweicloud.com/ces/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Resource Group](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ces/resource-group)
