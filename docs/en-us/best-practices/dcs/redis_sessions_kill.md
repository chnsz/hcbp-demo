# Deploy Redis Sessions Kill

## Application Scenario

During operation, a Distributed Cache Service (DCS) Redis instance establishes connections with multiple clients. When some clients become abnormal or leak connections, they occupy the connection resources of the instance and affect its stable operation. The sessions kill function can disconnect specified sessions in batches by client address, releasing connection resources in time.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis single instance and perform sessions kill based on the instance shard information, including VPC creation, subnet configuration, availability zone and flavor queries, instance configuration, shard query, and sessions kill.

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
- [DCS Sessions Kill (huaweicloud_dcs_sessions_kill)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_sessions_kill)

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
                └── huaweicloud_dcs_sessions_kill

random_password
    └── huaweicloud_dcs_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Virtual Private Cloud

Add the following script to the TF file (such as main.tf) to create a VPC:

```hcl
# Create a virtual private cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter description**:
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr

### 3. Create a Virtual Private Cloud Subnet

Add the following script to the TF file (such as main.tf) to create a subnet:

```hcl
# Create a virtual private cloud subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter description**:
- **vpc_id**: Assigned by referencing the ID of the resource huaweicloud_vpc.test
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: When the input variable subnet_cidr is empty, the subnet CIDR is automatically allocated based on the VPC CIDR block
- **gateway_ip**: When the input variable subnet_gateway_ip is empty, the first available IP of the subnet CIDR is used as the gateway address

### 4. Query the Availability Zones

Add the following script to the TF file (such as main.tf) to query the availability zones:

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

**Parameter description**:
- **count**: The query is executed when the input variable availability_zone is empty, otherwise it is not executed

### 5. Query the DCS Flavors

Add the following script to the TF file (such as main.tf) to query the DCS flavors:

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
  default     = 1
}

variable "instance_engine_version" {
  description = "The engine version of the Redis single instance"
  type        = string
  default     = "7.0"
}

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  cache_mode     = "single"
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**Parameter description**:
- **count**: The query is executed when the input variable instance_flavor_id is empty, otherwise it is not executed
- **cache_mode**: The cache mode, fixed to single
- **capacity**: Assigned by referencing the input variable instance_capacity
- **engine_version**: Assigned by referencing the input variable instance_engine_version

### 6. Create a Random Password

Add the following script to the TF file (such as main.tf) to create a random password:

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

**Parameter description**:
- **count**: The random password is created when the input variable instance_password is empty, otherwise it is not created
- **length**: The password length, fixed to 12
- **special**: Whether to include special characters, fixed to true
- **override_special**: The set of allowed special characters
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters respectively

### 7. Create a DCS Instance

Add the following script to the TF file (such as main.tf) to create a DCS Redis single instance:

```hcl
# Create a DCS instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the Redis single instance"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the Redis single instance belongs"
  type        = string
  default     = null
}

variable "charging_mode" {
  description = "The charging mode of the Redis instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The unit of the period"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the Redis instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "Whether auto renew is enabled"
  type        = string
  default     = "false"
}

resource "huaweicloud_dcs_instance" "test" {
  name                  = var.instance_name
  engine                = "Redis"
  enterprise_project_id = var.enterprise_project_id
  engine_version        = var.instance_engine_version
  capacity              = var.instance_capacity
  flavor                = (var.instance_flavor_id != "" ? var.instance_flavor_id :
  try(data.huaweicloud_dcs_flavors.test[0].flavors[0].name, null))
  availability_zones    = (var.availability_zone != "" ? [var.availability_zone] :
  try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null))
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  password              = (var.instance_password != "" ? var.instance_password :
  try(random_password.test[0].result, null))
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew

  lifecycle {
    ignore_changes = [
      flavor,
      availability_zones,
    ]
  }
}
```

**Parameter description**:
- **name**: Assigned by referencing the input variable instance_name
- **engine**: The cache engine, fixed to Redis
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id
- **engine_version**: Assigned by referencing the input variable instance_engine_version
- **capacity**: Assigned by referencing the input variable instance_capacity
- **flavor**: Uses the input variable instance_flavor_id when it is not empty, otherwise uses the queried flavor name
- **availability_zones**: Uses the input variable availability_zone when it is not empty, otherwise uses the first of the queried availability zone names
- **vpc_id**: Assigned by referencing the ID of the resource huaweicloud_vpc.test
- **subnet_id**: Assigned by referencing the ID of the resource huaweicloud_vpc_subnet.test
- **password**: Uses the input variable instance_password when it is not empty, otherwise uses the randomly generated password
- **charging_mode**, **period_unit**, **period**, **auto_renew**: Assigned by referencing the corresponding input variables

### 8. Query the DCS Instance Shards

Add the following script to the TF file (such as main.tf) to query the DCS instance shards:

```hcl
# Query the DCS instance shards in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_dcs_instance_shards" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
}

locals {
  replication_list = try(data.huaweicloud_dcs_instance_shards.test.group_list[0].replication_list, [])
}
```

**Parameter description**:
- **instance_id**: Assigned by referencing the ID of the resource huaweicloud_dcs_instance.test
- **locals.replication_list**: Extracts the replication list from the shard query result for obtaining the node ID later

### 9. Create a DCS Sessions Kill

Add the following script to the TF file (such as main.tf) to perform sessions kill:

```hcl
# Create a DCS sessions kill resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "client_addrs" {
  description = "The list of client addresses to be killed"
  type        = list(string)
  default     = ["127.0.0.1:6379"]
}

resource "huaweicloud_dcs_sessions_kill" "test" {
  instance_id  = huaweicloud_dcs_instance.test.id
  node_id      = try(local.replication_list[0].node_id, "")
  client_addrs = var.client_addrs
}
```

**Parameter description**:
- **instance_id**: Assigned by referencing the ID of the resource huaweicloud_dcs_instance.test
- **node_id**: Obtains the node ID of the first replication from the shard query result
- **client_addrs**: Assigned by referencing the input variable client_addrs, specifying the list of client addresses to be killed

### 10. Preset Input Parameters Required for Resource Deployment (Optional)

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

### 11. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DCS Redis instance and performing sessions kill
4. Run `terraform show` to view the created DCS Redis instance and the sessions kill result

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Sessions Kill](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-sessions-kill)
