# Deploy Redis Node Priority Config

## Application Scenario

A DCS Redis HA instance consists of a primary node and standby nodes. When the primary node fails, the system automatically performs a primary/standby switchover. The priority weight of a standby node determines the failover order when multiple standby nodes exist: the higher the weight, the more likely the node is promoted to primary.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis instance and configure the standby node priority weight, including VPC creation, subnet configuration, availability zone and flavor queries, instance creation, shard information query, and standby node priority weight configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS Flavors (data.huaweicloud_dcs_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)
- [DCS Instance Shards (data.huaweicloud_dcs_instance_shards)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_instance_shards)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DCS Instance (huaweicloud_dcs_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS Node Priority Config (huaweicloud_dcs_node_priority_config)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_node_priority_config)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
            └── data.huaweicloud_dcs_instance_shards
                └── huaweicloud_dcs_node_priority_config

random_password
    └── huaweicloud_dcs_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a VPC and Subnet

Add the following script to the TF file (such as main.tf) to create a VPC and subnet:

```hcl
# Create VPC and subnet resources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
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
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing `huaweicloud_vpc.test.id`
- **cidr**: The CIDR block of the subnet. When the input variable subnet_cidr is empty, it is automatically calculated based on the VPC CIDR block using the `cidrsubnet` function
- **gateway_ip**: The gateway IP address of the subnet. When the input variable subnet_gateway_ip is empty, it is automatically calculated based on the subnet CIDR block using the `cidrhost` function

### 3. Query Availability Zones and Flavors

Add the following script to the TF file (such as main.tf) to query availability zones and DCS flavors:

```hcl
# Query availability zones and DCS flavors data sources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone to which the Redis single instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_id" {
  description = "The flavor ID of the Redis single instance"
  type        = string
  default     = ""
}

variable "instance_capacity" {
  description = "The capacity of the Redis instance (in GB)"
  type        = number
  default     = 4
}

variable "instance_engine_version" {
  description = "The engine version of the Redis single instance"
  type        = string
  default     = "5.0"
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  engine         = "Redis"
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable availability_zone is empty, used to query the list of availability zones in the current region
- **count**: The data source is created when the input variable instance_flavor_id is empty, used to query DCS flavors that meet the conditions
- **engine**: The cache engine type, fixed to Redis
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version

### 4. Create a Random Password

Add the following script to the TF file (such as main.tf) to generate a random password:

```hcl
# Create a random password resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **count**: The resource is created when the input variable instance_password is empty, used to automatically generate the instance password
- **length**: The password length, set to 12
- **special**: Whether to include special characters, set to true
- **override_special**: The set of allowed special characters
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters respectively

### 5. Create a DCS Redis Instance

Add the following script to the TF file (such as main.tf) to create a DCS Redis instance:

```hcl
# Create a DCS instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **engine**: The cache engine type, fixed to Redis
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity
- **vpc_id**: The ID of the VPC to which the instance belongs, assigned by referencing `huaweicloud_vpc.test.id`
- **subnet_id**: The ID of the subnet to which the instance belongs, assigned by referencing `huaweicloud_vpc_subnet.test.id`
- **password**: The instance password. When the input variable instance_password is not empty, this value is used; otherwise, the randomly generated password is used
- **flavor**: The instance flavor. When the input variable instance_flavor_id is not empty, this value is used; otherwise, the queried flavor name is used
- **availability_zones**: The list of availability zones to which the instance belongs. When the input variable availability_zone is not empty, this value is used; otherwise, the first queried availability zone is used

### 6. Query DCS Instance Shards

Add the following script to the TF file (such as main.tf) to query the shard information of the DCS instance:

```hcl
# Query DCS instance shards data source in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_dcs_instance_shards" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
}

locals {
  replication_list = try(data.huaweicloud_dcs_instance_shards.test.group_list[0].replication_list, [])
}
```

**Parameter Description**:
- **instance_id**: The DCS instance ID, assigned by referencing `huaweicloud_dcs_instance.test.id`
- **replication_list**: A local variable used to extract the replication list of the first shard group, which is later used to obtain the standby node ID

### 7. Configure the Standby Node Priority Weight

Add the following script to the TF file (such as main.tf) to configure the standby node priority weight:

```hcl
# Create a DCS node priority config resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "slave_priority_weight" {
  description = "The slave node priority weight (0-100, 0 means failover is prohibited)"
  type        = number
  default     = 50
}

resource "huaweicloud_dcs_node_priority_config" "test" {
  instance_id           = huaweicloud_dcs_instance.test.id
  group_id              = try(data.huaweicloud_dcs_instance_shards.test.group_list[0].group_id, null)
  node_id               = try(local.replication_list[0].node_id, "")
  slave_priority_weight = var.slave_priority_weight
}
```

**Parameter Description**:
- **instance_id**: The DCS instance ID, assigned by referencing `huaweicloud_dcs_instance.test.id`
- **group_id**: The shard group ID, assigned by referencing the group_id of the first shard group in the data source `data.huaweicloud_dcs_instance_shards.test`
- **node_id**: The standby node ID, assigned by referencing the node_id of the first replication in the local variable `local.replication_list`
- **slave_priority_weight**: The standby node priority weight, ranging from 0 to 100, assigned by referencing the input variable slave_priority_weight, where 0 means failover is prohibited

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
vpc_name      = "tf_test_dcs_instance_vpc"
subnet_name   = "tf_test_dcs_instance_subnet"
instance_name = "tf_test_dcs_instance"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DCS Redis instance and configuring the standby node priority weight
4. Run `terraform show` to view the created DCS Redis instance and node priority config

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Node Priority Config](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-node-priority-config)
