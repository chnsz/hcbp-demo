# Deploy Redis Big Key Analysis

## Application Scenario

Distributed Cache Service (DCS) is a high-performance, highly available in-memory database service provided by Huawei Cloud, supporting mainstream cache engines such as Redis and Memcached. When large keys exist in a Redis instance, issues such as high memory usage, request timeouts, and blocking may occur, affecting business stability.

This best practice will introduce how to use Terraform to create a VPC, subnet, and DCS Redis instance, and configure a big key analysis task for it, helping you detect and handle large keys in a timely manner and ensure the stable running of the cache service.

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
- [DCS Big Key Analysis (huaweicloud_dcs_bigkey_analysis)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_bigkey_analysis)

### Resource/Data Source Dependencies

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
huaweicloud_availability_zones
huaweicloud_dcs_flavors
random_password
    └── huaweicloud_dcs_instance
        └── huaweicloud_dcs_bigkey_analysis
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified working directory, and ensure that it (or other TF files in the same level directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
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
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id.
- **name**: Assigned by referencing the input variable subnet_name.
- **cidr**: Uses the value of subnet_cidr if it is not empty; otherwise, it is automatically allocated from the VPC CIDR using the cidrsubnet function.
- **gateway_ip**: Uses the value of subnet_gateway_ip if it is not empty; otherwise, it is automatically calculated using the cidrhost function.

### 4. Query Availability Zones and DCS Flavors

Add the following script to the TF file (such as main.tf) to query available availability zones and DCS flavors:

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
- **huaweicloud_availability_zones**: When availability_zone is empty, queries the list of available availability zones.
- **huaweicloud_dcs_flavors**: When instance_flavor_id is empty, queries matching DCS flavors based on cache_mode, capacity, and engine_version.

### 5. Generate Random Password

Add the following script to the TF file (such as main.tf) to generate a random password when no instance password is specified:

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

locals {
  dcs_password = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, "")
}
```

**Parameter Description**:
- **random_password**: When instance_password is empty, generates a random password containing uppercase letters, lowercase letters, digits, and special characters.
- **locals.dcs_password**: Uses the user-provided password if available; otherwise, uses the generated random password.

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
  password           = local.dcs_password
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
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id.
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id.
- **password**: Assigned by referencing local.dcs_password.
- **flavor**: Uses the value of instance_flavor_id if it is not empty; otherwise, selects the first flavor from the queried flavor list.
- **availability_zones**: Uses the value of availability_zone if it is not empty; otherwise, selects the first availability zone from the queried list.

### 7. Create DCS Big Key Analysis

Add the following script to the TF file (such as main.tf) to create a DCS big key analysis task:

```hcl
# Create a DCS big key analysis in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_dcs_bigkey_analysis" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing huaweicloud_dcs_instance.test.id.

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this best practice, some resources and data sources use input variables to assign values to configurations. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through the `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory. The example content is as follows:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
vpc_name      = "tf_test_dcs_instance_vpc"
subnet_name   = "tf_test_dcs_instance_subnet"
instance_name = "tf_test_dcs_instance"
```

**Usage**:

1. Save the above content as the `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DCS Redis instance and big key analysis task
4. Run `terraform show` to view the created DCS Redis instance and big key analysis task

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Big Key Analysis](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-bigkey-analysis)
