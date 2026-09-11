# Deploy Redis Expired Key Scan

## Application Scenario

Distributed Cache Service (DCS) is a high-performance, highly available in-memory database service provided by Huawei Cloud, supporting mainstream cache engines such as Redis. As services run, a large number of expired keys accumulate in Redis instances. If these keys are not cleaned up for a long time, they occupy memory resources and affect instance performance.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis instance and trigger an expired key scan, including VPC creation, subnet configuration, instance creation, and expired key scan task triggering, helping you keep track of the number of expired keys in the instance and providing a basis for subsequent memory optimization and operation work.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS Flavors (data.huaweicloud_dcs_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DCS Instance (huaweicloud_dcs_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS Instance Expired Key Scan (huaweicloud_dcs_instance_expired_key_scan)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance_expired_key_scan)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
            └── huaweicloud_dcs_instance_expired_key_scan

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
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr, with a default value of 192.168.0.0/16

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
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, which is the ID of the VPC created in the previous step
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; if empty, the subnet CIDR is automatically calculated based on the VPC CIDR
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip; if empty, the gateway IP is automatically calculated based on the subnet CIDR

### 4. Query Availability Zones and DCS Flavors

Add the following script in the TF file (such as main.tf) to query availability zones and DCS flavors:

```hcl
# Query availability zones and DCS flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  default     = 1
}

variable "instance_engine_version" {
  description = "The engine version of the Redis single instance"
  type        = string
  default     = "6.0"
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
- **count**: Queries the availability zone list when the input variable availability_zone is empty, otherwise does not query
- **engine**: The cache engine of the DCS flavor, fixed to Redis
- **capacity**: Assigned by referencing the input variable instance_capacity, indicating the instance capacity (in GB)
- **engine_version**: Assigned by referencing the input variable instance_engine_version, indicating the instance engine version

### 5. Create a Random Password

Add the following script in the TF file (such as main.tf) to automatically generate a random password when no password is specified:

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
- **count**: Generates a random password when the input variable instance_password is empty, otherwise does not generate
- **length**: The length of the random password, fixed to 12
- **special**: Whether to include special characters, fixed to true
- **override_special**: The set of allowed special characters
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters respectively

### 6. Create a DCS Redis Instance

Add the following script in the TF file (such as main.tf) to create a DCS Redis instance:

```hcl
# Create DCS Redis instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **name**: Assigned by referencing the input variable instance_name
- **engine**: The cache engine, fixed to Redis
- **engine_version**: Assigned by referencing the input variable instance_engine_version
- **capacity**: Assigned by referencing the input variable instance_capacity
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id
- **password**: Assigned by referencing the input variable instance_password; if empty, the randomly generated password is used
- **flavor**: Assigned by referencing the input variable instance_flavor_id; if empty, the queried DCS flavor name is used
- **availability_zones**: Assigned by referencing the input variable availability_zone; if empty, the first queried availability zone is used

### 7. Create a DCS Instance Expired Key Scan

Add the following script in the TF file (such as main.tf) to create a DCS instance expired key scan:

```hcl
# Create DCS instance expired key scan resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_dcs_instance_expired_key_scan" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing huaweicloud_dcs_instance.test.id, which is the ID of the DCS Redis instance created in the previous step

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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DCS Redis instance and the expired key scan task
4. Run `terraform show` to view the created DCS Redis instance and the expired key scan task

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Expired Key Scan](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-expired-key-scan)
