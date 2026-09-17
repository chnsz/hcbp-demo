# Deploy RabbitMQ Basic Instance

## Application Scenario

Distributed Message Service (DMS) RabbitMQ is a high-performance, highly reliable message middleware service based on the AMQP protocol, widely used in scenarios such as asynchronous communication, system decoupling, and traffic peak shaving. With RabbitMQ instances, enterprises can build loosely coupled and scalable distributed application architectures, ensuring reliable message transmission and efficient flow.

This best practice will introduce how to use Terraform to automatically deploy a DMS RabbitMQ basic instance, including the creation of VPC, subnet, security group, and RabbitMQ instance.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RabbitMQ Instance Flavors (data.huaweicloud_dms_rabbitmq_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_rabbitmq_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [RabbitMQ Instance (huaweicloud_dms_rabbitmq_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_instance)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dms_rabbitmq_instance

data.huaweicloud_dms_rabbitmq_flavors
    └── huaweicloud_dms_rabbitmq_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dms_rabbitmq_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dms_rabbitmq_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf) to query availability zones:

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zones" {
  description = "The availability zones to which the RabbitMQ instance belongs"
  type        = list(string)
  default     = []
}

data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable availability_zones is empty, used to automatically obtain the availability zone list in the current region

### 3. Create a VPC

Add the following script in the TF file (such as main.tf) to create a VPC:

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

### 4. Create a VPC Subnet

Add the following script in the TF file (such as main.tf) to create a VPC subnet:

```hcl
# Create a VPC subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "192.168.0.0/24"
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = "192.168.0.1"
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing the ID of the VPC resource
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; if empty, it is automatically calculated based on the VPC CIDR
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip; if empty, it is automatically calculated based on the VPC CIDR

### 5. Create a Security Group

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
- **name**: Assigned by referencing the input variable security_group_name
- **delete_default_rules**: Set to true to delete the default rules of the security group

### 6. Query RabbitMQ Instance Flavors

Add the following script in the TF file (such as main.tf) to query RabbitMQ instance flavors:

```hcl
# Query the RabbitMQ instance flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_flavor_id" {
  description = "The flavor ID of the RabbitMQ instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_type" {
  description = "The flavor type of the RabbitMQ instance"
  type        = string
  default     = "cluster"
}

variable "instance_storage_spec_code" {
  description = "The storage specification code of the RabbitMQ instance"
  type        = string
  default     = "dms.physical.storage.ultra.v2"
}

variable "availability_zone_number" {
  description = "The number of availability zones to which the RabbitMQ instance belongs"
  type        = number
  default     = 1
}

data "huaweicloud_dms_rabbitmq_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  type               = var.instance_flavor_type
  flavor_id          = var.instance_flavor_id
  storage_spec_code  = var.instance_storage_spec_code
  availability_zones = length(var.availability_zones) != 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, var.availability_zone_number), null)
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable instance_flavor_id is empty, used to automatically obtain available RabbitMQ instance flavors
- **type**: Assigned by referencing the input variable instance_flavor_type; valid values are single or cluster
- **flavor_id**: Assigned by referencing the input variable instance_flavor_id
- **storage_spec_code**: Assigned by referencing the input variable instance_storage_spec_code
- **availability_zones**: Assigned by referencing the input variable availability_zones; if empty, it is automatically sliced based on the availability zone list and the input variable availability_zone_number

### 7. Create a RabbitMQ Instance

Add the following script in the TF file (such as main.tf) to create a RabbitMQ instance:

```hcl
# Create a RabbitMQ instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the RabbitMQ instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the RabbitMQ instance"
  type        = string
  default     = "3.8.35"
}

variable "instance_broker_num" {
  description = "The number of brokers of the RabbitMQ instance"
  type        = number
  default     = 3
}

variable "instance_storage_space" {
  description = "The storage space of the RabbitMQ instance"
  type        = number
  default     = 600
}

variable "instance_ssl_enable" {
  description = "The SSL enable of the RabbitMQ instance"
  type        = bool
  default     = false
}

variable "instance_access_user_name" {
  description = "The access user of the RabbitMQ instance"
  type        = string
}

variable "instance_password" {
  description = "The access password of the RabbitMQ instance"
  sensitive   = true
  type        = string
}

variable "instance_description" {
  description = "The description of the RabbitMQ instance"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the RabbitMQ instance belongs"
  type        = string
  default     = null
}

variable "instance_tags" {
  description = "The key/value pairs to associate with the instance"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the RabbitMQ instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the RabbitMQ instance"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the RabbitMQ instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the RabbitMQ instance"
  type        = string
  default     = "false"
}

resource "huaweicloud_dms_rabbitmq_instance" "test" {
  name                  = var.instance_name
  engine_version        = var.instance_engine_version
  flavor_id             = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_dms_rabbitmq_flavors.test[0].flavors[0].id, null)
  vpc_id                = huaweicloud_vpc.test.id
  network_id            = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zones    = length(var.availability_zones) != 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, var.availability_zone_number), null)
  broker_num            = var.instance_broker_num
  storage_space         = var.instance_storage_space
  storage_spec_code     = var.instance_storage_spec_code
  ssl_enable            = var.instance_ssl_enable
  access_user           = var.instance_access_user_name
  password              = var.instance_password
  description           = var.instance_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.instance_tags
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew

  # If you want to change some of the following parameters, you need to remove the corresponding fields from "lifecycle.ignore_changes".
  lifecycle {
    ignore_changes = [
      flavor_id,
      availability_zones,
    ]
  }
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable instance_name
- **engine_version**: Assigned by referencing the input variable instance_engine_version
- **flavor_id**: Assigned by referencing the input variable instance_flavor_id; if empty, the first flavor ID queried by the data source is used
- **vpc_id**: Assigned by referencing the ID of the VPC resource
- **network_id**: Assigned by referencing the ID of the VPC subnet resource
- **security_group_id**: Assigned by referencing the ID of the security group resource
- **availability_zones**: Assigned by referencing the input variable availability_zones; if empty, it is automatically sliced based on the availability zone list and the input variable availability_zone_number
- **broker_num**: Assigned by referencing the input variable instance_broker_num; for a single instance, this value can only be 1
- **storage_space**: Assigned by referencing the input variable instance_storage_space
- **storage_spec_code**: Assigned by referencing the input variable instance_storage_spec_code
- **ssl_enable**: Assigned by referencing the input variable instance_ssl_enable
- **access_user**: Assigned by referencing the input variable instance_access_user_name
- **password**: Assigned by referencing the input variable instance_password
- **description**: Assigned by referencing the input variable instance_description
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id
- **tags**: Assigned by referencing the input variable instance_tags
- **charging_mode**: Assigned by referencing the input variable charging_mode
- **period_unit**: Assigned by referencing the input variable period_unit
- **period**: Assigned by referencing the input variable period
- **auto_renew**: Assigned by referencing the input variable auto_renew

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name                  = "tf_test_rabbitmq_vpc"
subnet_name               = "tf_test_rabbitmq_subnet"
security_group_name       = "tf_test_rabbitmq_security_group"
instance_name             = "tf_test_rabbitmq"
instance_access_user_name = "tf_test_rabbitmq_user"
instance_password         = "YourPassword!"
instance_tags = {
  owner = "terraform"
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

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the RabbitMQ instance
4. Run `terraform show` to view the created RabbitMQ instance

## Reference Information

- [Huawei Cloud Distributed Message Service RabbitMQ Product Documentation](https://support.huaweicloud.com/rabbitmq/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DMS RabbitMQ Basic Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/rabbitmq/basic-instance)
