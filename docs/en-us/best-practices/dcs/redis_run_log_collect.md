# Deploy Redis Run Log Collect

## Application Scenario

A DCS Redis instance generates run logs during operation, recording key information such as running status, slow queries, and connection exceptions. When an instance experiences performance fluctuations or exceptions, O&M personnel need to collect run logs within a specified time range for analysis and troubleshooting.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis HA instance and collect its run logs, including VPC and subnet creation, availability zone and flavor queries, instance configuration, node query, and run log collection task creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS Flavors (data.huaweicloud_dcs_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)
- [DCS Instance Nodes (data.huaweicloud_dcs_instance_nodes)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_instance_nodes)

### Resources

- [VPC (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DCS Instance (huaweicloud_dcs_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS Redis Run Log Collect (huaweicloud_dcs_redis_run_log_collect)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_redis_run_log_collect)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
            └── data.huaweicloud_dcs_instance_nodes
                └── huaweicloud_dcs_redis_run_log_collect

random_password
    └── huaweicloud_dcs_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

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
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr

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
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The subnet CIDR block; when the input variable subnet_cidr is empty, it is automatically divided from the VPC CIDR block
- **gateway_ip**: The subnet gateway IP; when the input variable subnet_gateway_ip is empty, it is automatically calculated from the subnet CIDR block

### 4. Query the Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones:

```hcl
# Query availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **count**: The query is executed when the input variable availability_zone is empty; otherwise, the user-specified availability zone is used

### 5. Query the DCS Flavors

Add the following script in the TF file (such as main.tf) to query the DCS flavors:

```hcl
# Query DCS flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  cache_mode       = "ha"
  cpu_architecture = "x86_64"
  capacity         = var.instance_capacity
}
```

**Parameter Description**:
- **count**: The query is executed when the input variable instance_flavor_id is empty; otherwise, the user-specified flavor is used
- **cache_mode**: The cache instance type, fixed to `ha` (master-standby)
- **cpu_architecture**: The CPU architecture, fixed to `x86_64`
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity

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
- **count**: A random password is generated when the input variable instance_password is empty; otherwise, the user-specified password is used
- **length**: The password length, fixed to 12
- **special**: Whether to include special characters, fixed to `true`
- **override_special**: The set of allowed special characters
- **min_upper**/**min_lower**/**min_numeric**/**min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters

### 7. Create a DCS Redis Instance

Add the following script in the TF file (such as main.tf) to create a DCS Redis instance:

```hcl
# Create DCS instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the Redis single instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the Redis single instance"
  type        = string
  default     = "5.0"
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
- **engine**: The cache engine, fixed to `Redis`
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC created in the previous step
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet created in the previous step
- **password**: The instance password; when the input variable instance_password is empty, the randomly generated password is used
- **flavor**: The instance flavor; when the input variable instance_flavor_id is empty, the queried flavor is used
- **availability_zones**: The availability zones to which the instance belongs; when the input variable availability_zone is empty, the queried availability zones are used

### 8. Query the DCS Instance Nodes

Add the following script in the TF file (such as main.tf) to query the DCS instance nodes:

```hcl
# Query DCS instance nodes in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_dcs_instance_nodes" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
}
```

**Parameter Description**:
- **instance_id**: The instance ID, referencing the ID of the DCS instance created in the previous step

### 9. Create a Redis Run Log Collect Task

Add the following script in the TF file (such as main.tf) to create a Redis run log collect task:

```hcl
# Create Redis run log collect resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "log_type" {
  description = "The type of log to collect. Currently, only run (Redis running log) is supported"
  type        = string
  default     = "run"
}

variable "query_time" {
  description = "The date offset in days. Valid values: 0, 1, 3, 7. 0 means today's logs"
  type        = number
  default     = 0
}

resource "huaweicloud_dcs_redis_run_log_collect" "test" {
  instance_id    = huaweicloud_dcs_instance.test.id
  log_type       = var.log_type
  query_time     = var.query_time
  replication_id = try(data.huaweicloud_dcs_instance_nodes.test.nodes[0].replication_id, "")
}
```

**Parameter Description**:
- **instance_id**: The instance ID, referencing the ID of the DCS instance created in the previous step
- **log_type**: The log type, assigned by referencing the input variable log_type; currently only `run` (Redis running log) is supported
- **query_time**: The date offset in days, assigned by referencing the input variable query_time; valid values are 0, 1, 3, and 7, where 0 means today's logs
- **replication_id**: The replication ID, referencing the replication ID of the queried instance node

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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Redis run log collect task
4. Run `terraform show` to view the created Redis run log collect task

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Run Log Collect](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-run-log-collect)
