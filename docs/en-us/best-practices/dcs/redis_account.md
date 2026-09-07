# Deploy Redis Account Management

## Application Scenario

Distributed Cache Service (DCS) is a high-performance, highly available in-memory database service provided by Huawei Cloud, supporting mainstream cache engines such as Redis and Memcached. In real business scenarios, to meet the isolation access requirements of multiple users and applications, it is usually necessary to create independent accounts on DCS Redis instances and assign read-only or read-write permissions to them, so as to implement fine-grained access control.

This best practice will introduce how to use Terraform to automate the deployment of DCS Redis instances and their accounts, including creating a VPC, subnet, Redis instance, and DCS account, and configuring the corresponding access permissions.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS Flavors (huaweicloud_dcs_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DCS Redis Instance (huaweicloud_dcs_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS Account (huaweicloud_dcs_account)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_account)

### Resource/Data Source Dependencies

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
          └── huaweicloud_dcs_instance
                └── huaweicloud_dcs_account

data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

random_password
    └── huaweicloud_dcs_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF files (such as main.tf) required for writing the current best practice script in the specified working directory, and ensure that they (or other TF files in the same level directory) contain the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create VPC

Add the following script to the TF file (such as main.tf) to create a VPC:

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

- **name**: Assigned by referencing the input variable vpc_name.
- **cidr**: Assigned by referencing the input variable vpc_cidr.

### 3. Create Subnet

Add the following script to the TF file (such as main.tf) to create a subnet:

```hcl
# Create a subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

- **vpc_id**: Assigned by referencing the ID of the created huaweicloud_vpc.test.
- **name**: Assigned by referencing the input variable subnet_name.
- **cidr**: Assigned by referencing the input variable subnet_cidr. If not specified, it is automatically divided from the VPC CIDR.
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip. If not specified, it is automatically calculated.

### 4. Query Availability Zones and DCS Flavors

Add the following script to the TF file (such as main.tf) to query availability zones and DCS flavors:

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

- **availability_zone**: Assigned by referencing the input variable availability_zone. If not specified, availability zones are queried.
- **instance_flavor_id**: Assigned by referencing the input variable instance_flavor_id. If not specified, flavors are queried.
- **cache_mode**: Fixed to "ha", indicating master-standby mode.
- **capacity**: Assigned by referencing the input variable instance_capacity.
- **engine_version**: Assigned by referencing the input variable instance_engine_version.

### 5. Generate Random Password

Add the following script to the TF file (such as main.tf) to generate a random password when the instance password is not specified:

```hcl
# Generate a random password
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

- **length**: Password length, fixed to 12.
- **special**: Whether to include special characters, fixed to true.
- **override_special**: Specifies the set of special characters available.
- **min_upper**: Minimum number of uppercase letters.
- **min_lower**: Minimum number of lowercase letters.
- **min_numeric**: Minimum number of digits.
- **min_special**: Minimum number of special characters.

### 6. Create DCS Redis Instance

Add the following script to the TF file (such as main.tf) to create a DCS Redis instance:

```hcl
# Create a DCS Redis instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

- **name**: Assigned by referencing the input variable instance_name.
- **engine**: Fixed to "Redis".
- **engine_version**: Assigned by referencing the input variable instance_engine_version.
- **capacity**: Assigned by referencing the input variable instance_capacity.
- **vpc_id**: Assigned by referencing the ID of the created huaweicloud_vpc.test.
- **subnet_id**: Assigned by referencing the ID of the created huaweicloud_vpc_subnet.test.
- **password**: Assigned by referencing the input variable instance_password. If not specified, the password generated by random_password is used.
- **flavor**: Assigned by referencing the input variable instance_flavor_id. If not specified, the queried flavor is used.
- **availability_zones**: Assigned by referencing the input variable availability_zone. If not specified, the queried availability zones are used.

### 7. Create DCS Account

Add the following script to the TF file (such as main.tf) to create a DCS account:

```hcl
# Create a DCS account in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "account_name" {
  description = "The name of the DCS account"
  type        = string
}

variable "account_role" {
  description = "The role of the DCS account. Valid values: read, write"
  type        = string
  default     = "read"
}

variable "account_password" {
  description = "The password of the DCS account"
  type        = string
  sensitive   = true
}

variable "account_description" {
  description = "The description of the DCS account"
  type        = string
  default     = ""
}

resource "huaweicloud_dcs_account" "test" {
  instance_id      = huaweicloud_dcs_instance.test.id
  account_name     = var.account_name
  account_role     = var.account_role
  account_password = var.account_password
  description      = var.account_description
}
```

**Parameter Description**:

- **instance_id**: Assigned by referencing the ID of the created huaweicloud_dcs_instance.test.
- **account_name**: Assigned by referencing the input variable account_name.
- **account_role**: Assigned by referencing the input variable account_role. Valid values are read and write.
- **account_password**: Assigned by referencing the input variable account_password.
- **description**: Assigned by referencing the input variable account_description.

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this best practice, some resources and data sources use input variables to assign values to configurations. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory. The example content is as follows:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
vpc_name         = "tf_test_dcs_instance_vpc2"
subnet_name      = "tf_test_dcs_instance_subnet"
instance_name    = "tf_test_dcs_instance"
account_name     = "tf_test_account"
account_password = "Terraform@123"
```

**Usage**:

1. Save the above content as the `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DCS Redis instance and account
4. Run `terraform show` to view the created DCS Redis instance and account

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Account Management](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-account)
