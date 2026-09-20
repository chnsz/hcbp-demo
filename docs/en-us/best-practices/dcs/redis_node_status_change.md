# Deploy Redis Node Status Change

## Application Scenario

During the operation of a Distributed Cache Service (DCS) Redis instance, nodes may need to be started or stopped for reasons such as node maintenance, troubleshooting, or resource adjustment. Huawei Cloud DCS provides the node status change capability, which supports starting or stopping specified nodes to meet daily O&M and fault handling requirements.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis instance, create a backup before changing the node status to ensure data safety, and finally start or stop instance nodes through the node status change resource.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS Flavors (data.huaweicloud_dcs_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DCS Instance (huaweicloud_dcs_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS Backup (huaweicloud_dcs_backup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_backup)
- [DCS Node Status Change (huaweicloud_dcs_node_status_change)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_node_status_change)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
            ├── huaweicloud_dcs_backup
            └── huaweicloud_dcs_node_status_change

random_password
    └── huaweicloud_dcs_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a VPC

Add the following script in the TF file (such as main.tf) to create a VPC:

```hcl
# Create VPC resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr

### 3. Create a Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  gateway_ip = (var.subnet_gateway_ip != "" ? var.subnet_gateway_ip :
  cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1))
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC resource created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet, automatically calculated based on the VPC CIDR block when the input variable subnet_cidr is empty
- **gateway_ip**: The gateway IP address of the subnet, automatically calculated based on the subnet CIDR block when the input variable subnet_gateway_ip is empty

### 4. Query the Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones:

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone to which the Redis single instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: The query is executed when the input variable availability_zone is empty, otherwise it is not executed

### 5. Query the DCS Flavors

Add the following script in the TF file (such as main.tf) to query the DCS flavors:

```hcl
# Query the DCS flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_flavor_id" {
  description = "The flavor ID of the Redis single instance"
  type        = string
  default     = ""
}

variable "instance_capacity" {
  description = "The capacity of the Redis instance (in GB)"
  type        = number
  default     = 0.125
}

variable "instance_engine_version" {
  description = "The engine version of the Redis single instance"
  type        = string
  default     = "5.0"
}

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  cache_mode     = "ha"
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**Parameter Description**:
- **count**: The query is executed when the input variable instance_flavor_id is empty, otherwise it is not executed
- **cache_mode**: The cache mode, fixed to ha here
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version

### 6. Create a Random Password

Add the following script in the TF file (such as main.tf) to create a random password:

```hcl
# Create random password resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_password" {
  description = "The password for the Redis instance"
  type        = string
  sensitive   = true
  default     = null
}

resource "random_password" "test" {
  count = var.instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "!@%^*-_=+"
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**Parameter Description**:
- **count**: The random password is created when the input variable instance_password is empty, otherwise it is not created
- **length**: The password length, fixed to 12 here
- **special**: Whether to include special characters, fixed to true here
- **override_special**: The set of allowed special characters
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters respectively

### 7. Create a DCS Redis Instance

Add the following script in the TF file (such as main.tf) to create a DCS Redis instance:

```hcl
# Create DCS instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the Redis single instance"
  type        = string
}

resource "huaweicloud_dcs_instance" "test" {
  name               = var.instance_name
  engine             = "Redis"
  engine_version     = var.instance_engine_version
  capacity           = var.instance_capacity
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  password           = (var.instance_password != "" ? var.instance_password :
  try(random_password.test[0].result, null))
  flavor             = (var.instance_flavor_id != "" ? var.instance_flavor_id :
  try(data.huaweicloud_dcs_flavors.test[0].flavors[0].name, null))
  availability_zones = (var.availability_zone != "" ? [var.availability_zone] :
  try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null))
}
```

**Parameter Description**:
- **name**: The instance name, assigned by referencing the input variable instance_name
- **engine**: The cache engine, fixed to Redis here
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC resource created in the previous step
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet resource created in the previous step
- **password**: The instance password, using the input variable instance_password when it is not empty, otherwise using the result generated by the random password resource
- **flavor**: The instance flavor, using the input variable instance_flavor_id when it is not empty, otherwise using the queried flavor name
- **availability_zones**: The list of availability zones to which the instance belongs, using the input variable availability_zone when it is not empty, otherwise using the first item of the queried availability zone list

### 8. Create a DCS Backup

Add the following script in the TF file (such as main.tf) to create a DCS backup:

```hcl
# Create DCS backup resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "backup_description" {
  description = "The description of the DCS backup"
  type        = string
  default     = "test DCS backup remark"
}

variable "backup_format" {
  description = "The format of the DCS backup. Valid values: rdb, aof"
  type        = string
  default     = "rdb"
}

resource "huaweicloud_dcs_backup" "test" {
  instance_id   = huaweicloud_dcs_instance.test.id
  description   = var.backup_description
  backup_format = var.backup_format
}
```

**Parameter Description**:
- **instance_id**: The ID of the instance to which the backup belongs, referencing the ID of the DCS instance resource created in the previous step
- **description**: The backup description, assigned by referencing the input variable backup_description
- **backup_format**: The backup format, assigned by referencing the input variable backup_format, with valid values rdb and aof

### 9. Change the DCS Instance Node Status

Add the following script in the TF file (such as main.tf) to change the DCS instance node status:

```hcl
# Create DCS node status change resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "action" {
  description = "The operation to perform on the instance nodes. Valid values: start, stop"
  type        = string
  default     = "start"
}

resource "huaweicloud_dcs_node_status_change" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
  action      = var.action
}
```

**Parameter Description**:
- **instance_id**: The ID of the instance whose node status is to be changed, referencing the ID of the DCS instance resource created in the previous step
- **action**: The operation to perform on the instance nodes, assigned by referencing the input variable action, with valid values start and stop

### 10. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# VPC and subnet configuration
vpc_name    = "tf_test_dcs_instance_vpc"
subnet_name = "tf_test_dcs_instance_subnet"

# DCS instance configuration
instance_name = "tf_test_dcs_instance"

# Node status change operation
action = "stop"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 11. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DCS Redis instance and changing the node status
4. Run `terraform show` to view the created DCS Redis instance and the node status change result

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Node Status Change](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-node-status-change)
