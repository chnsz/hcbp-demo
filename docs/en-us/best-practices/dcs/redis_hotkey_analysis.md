# Deploy Redis Hot Key Analysis

## Application Scenario

During the operation of a Distributed Cache Service (DCS) Redis instance, some keys may be accessed at a very high frequency, forming hot keys. Hot keys can cause excessive load on a single shard or node, degrading the response performance of the entire instance and even causing business jitter. Hot key analysis identifies the keys that are accessed most frequently in an instance, providing a basis for business optimization, shard adjustment, and cache policy improvement.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis instance and create a hot key analysis task on it, including VPC creation, subnet configuration, availability zone and flavor queries, instance creation, and hot key analysis task creation.

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
- [DCS Hot Key Analysis (huaweicloud_dcs_hotkey_analysis)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_hotkey_analysis)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
            └── huaweicloud_dcs_hotkey_analysis

random_password
    └── huaweicloud_dcs_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Virtual Private Cloud and Subnet

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
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr, defaulting to `192.168.0.0/16`
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing `huaweicloud_vpc.test.id`
- **cidr** (subnet): The subnet CIDR block; when the input variable subnet_cidr is empty, it is automatically calculated based on the VPC CIDR block using the `cidrsubnet` function
- **gateway_ip**: The subnet gateway IP; when the input variable subnet_gateway_ip is empty, it is automatically calculated based on the subnet CIDR block using the `cidrhost` function

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

- **count** (availability zones): Queries the availability zone list when the input variable availability_zone is empty
- **cache_mode**: The cache mode, fixed to `ha` (master-standby mode)
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity, defaulting to `0.125` GB
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version, defaulting to `5.0`

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
```

**Parameter Description**:

- **count**: Creates a random password when the input variable instance_password is empty
- **length**: The password length, which is `12` here
- **special**: Whether to include special characters, which is `true` here
- **override_special**: The set of allowed special characters
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters, respectively

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
  password           = (var.instance_password != "" ? var.instance_password :
  try(random_password.test[0].result, null))
  flavor             = (var.instance_flavor_id != "" ? var.instance_flavor_id :
  try(data.huaweicloud_dcs_flavors.test[0].flavors[0].name, null))
  availability_zones = (var.availability_zone != "" ? [var.availability_zone] :
  try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null))

  parameters {
    id    = "2"
    name  = "maxmemory-policy"
    value = "volatile-lfu"
  }
}
```

**Parameter Description**:

- **name**: The instance name, assigned by referencing the input variable instance_name
- **engine**: The cache engine, fixed to `Redis`
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity
- **vpc_id**: The ID of the VPC to which the instance belongs, assigned by referencing `huaweicloud_vpc.test.id`
- **subnet_id**: The ID of the subnet to which the instance belongs, assigned by referencing `huaweicloud_vpc_subnet.test.id`
- **password**: The instance password; when the input variable instance_password is not empty, this value is used; otherwise, the randomly generated password is used
- **flavor**: The instance flavor; when the input variable instance_flavor_id is not empty, this value is used; otherwise, the queried flavor name is used
- **availability_zones**: The availability zones of the instance; when the input variable availability_zone is not empty, this value is used; otherwise, the first item of the queried availability zone list is used
- **parameters**: The instance parameter configuration, which sets `maxmemory-policy` to `volatile-lfu`

### 6. Create a DCS Hot Key Analysis Task

Add the following script to the TF file (such as main.tf) to create a DCS hot key analysis task:

```hcl
# Create a DCS hot key analysis task resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_dcs_hotkey_analysis" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
}
```

**Parameter Description**:

- **instance_id**: The ID of the DCS instance on which hot key analysis is performed, assigned by referencing `huaweicloud_dcs_instance.test.id`

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

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

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DCS Redis instance and hot key analysis task
4. Run `terraform show` to view the created DCS Redis instance and hot key analysis task

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Hot Key Analysis](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-hotkey-analysis)
