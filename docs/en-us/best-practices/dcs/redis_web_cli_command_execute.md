# Deploy Redis Web CLI Command Execution

## Application Scenario

Distributed Cache Service (DCS) provides the Web CLI capability, which allows users to connect to a Redis instance and execute commands directly in a browser through the console or API, without installing client tools such as redis-cli locally. In automated O&M scenarios, you can use Terraform to log in to the Web CLI of an instance and then execute specified Redis commands, enabling online query and O&M operations on cached data.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis instance, log in to its Web CLI, and execute a specified Redis command, including VPC and subnet creation, availability zone and flavor query, instance creation, Web CLI login, and command execution.

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
- [DCS Web CLI Login (huaweicloud_dcs_login_web_cli)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_login_web_cli)
- [DCS Web CLI Command Execution (huaweicloud_dcs_web_cli_command_execute)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_web_cli_command_execute)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

random_password
    └── huaweicloud_dcs_instance
        └── huaweicloud_dcs_login_web_cli
            └── huaweicloud_dcs_web_cli_command_execute

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a VPC and a Subnet

Add the following script to the TF file (such as main.tf) to create a VPC and a subnet:

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
- **cidr**: The CIDR block of the subnet. When the input variable subnet_cidr is empty, it is automatically divided based on the VPC CIDR block using the `cidrsubnet` function
- **gateway_ip**: The gateway IP address of the subnet. When the input variable subnet_gateway_ip is empty, it is automatically calculated based on the subnet CIDR block using the `cidrhost` function

### 3. Query Availability Zones and DCS Flavors

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
  default     = 0.125
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

  cache_mode     = "ha"
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**Parameter Description**:
- **count**: Queries the availability zone list when the input variable availability_zone is empty
- **cache_mode**: The cache instance type, fixed to `ha` (master/standby) here
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version

### 4. Create a Random Password

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

locals {
  dcs_password = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, "")
}
```

**Parameter Description**:
- **count**: Generates a random password when the input variable instance_password is empty
- **length**: The password length, set to 12 here
- **special**: Whether to include special characters, set to `true` here
- **override_special**: The set of allowed special characters
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: Specify the minimum number of uppercase letters, lowercase letters, digits, and special characters respectively
- **dcs_password**: A local variable that uses the input variable instance_password when it is not empty, otherwise uses the randomly generated password

### 5. Create a DCS Redis Instance

Add the following script to the TF file (such as main.tf) to create a DCS Redis instance:

```hcl
# Create a DCS Redis instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  password           = local.dcs_password
  flavor             = (var.instance_flavor_id != "" ? var.instance_flavor_id :
  try(data.huaweicloud_dcs_flavors.test[0].flavors[0].name, null))
  availability_zones = (var.availability_zone != "" ? [var.availability_zone] :
  try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null))
}
```

**Parameter Description**:
- **name**: The instance name, assigned by referencing the input variable instance_name
- **engine**: The cache engine, fixed to `Redis` here
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity
- **vpc_id**: The ID of the VPC to which the instance belongs, assigned by referencing `huaweicloud_vpc.test.id`
- **subnet_id**: The ID of the subnet to which the instance belongs, assigned by referencing `huaweicloud_vpc_subnet.test.id`
- **password**: The instance password, assigned by referencing the local variable dcs_password
- **flavor**: The instance flavor. When the input variable instance_flavor_id is not empty, it uses that value; otherwise it uses the queried flavor name
- **availability_zones**: The availability zones of the instance. When the input variable availability_zone is not empty, it uses that value; otherwise it uses the first queried availability zone

### 6. Log In to the Web CLI

Add the following script to the TF file (such as main.tf) to log in to the Web CLI of the DCS instance:

```hcl
# Log in to the Web CLI of the DCS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_dcs_login_web_cli" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
  password    = local.dcs_password
}
```

**Parameter Description**:
- **instance_id**: The DCS instance ID, assigned by referencing `huaweicloud_dcs_instance.test.id`
- **password**: The instance password, assigned by referencing the local variable dcs_password

### 7. Execute a Web CLI Command

Add the following script to the TF file (such as main.tf) to execute a Web CLI command:

```hcl
# Execute a Web CLI command in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "command" {
  description = "The Redis command to be executed via Web CLI"
  type        = string
  default     = "scan 0"
}

variable "database" {
  description = "The database number to execute the command on"
  type        = number
  default     = 0
}

resource "huaweicloud_dcs_web_cli_command_execute" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
  client_id   = huaweicloud_dcs_login_web_cli.test.client_id
  command     = var.command
  database    = var.database
}
```

**Parameter Description**:
- **instance_id**: The DCS instance ID, assigned by referencing `huaweicloud_dcs_instance.test.id`
- **client_id**: The Web CLI client ID, assigned by referencing `huaweicloud_dcs_login_web_cli.test.client_id`, no manual configuration is required
- **command**: The Redis command to be executed, assigned by referencing the input variable command
- **database**: The database number to execute the command on, assigned by referencing the input variable database

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

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

# Web CLI command configuration
command  = "scan 0"
database = 0
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DCS Redis instance and executing the Web CLI command
4. Run `terraform show` to view the created DCS Redis instance and the Web CLI command execution result

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Web CLI Command Execution](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-web-cli-command-execute)
