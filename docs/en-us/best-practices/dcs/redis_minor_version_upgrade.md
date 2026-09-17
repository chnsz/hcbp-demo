# Deploy Redis Instance Minor Version Upgrade

## Application Scenario

During long-term operation of a Distributed Cache Service (DCS) Redis instance, Huawei Cloud continuously releases minor versions of the engine and proxy components to fix known defects and improve stability and performance. When the minor version of an instance lags behind the latest version, you can upgrade the engine and proxy components to the target version through a minor version upgrade, thereby obtaining better reliability and security.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis instance and trigger a minor version upgrade for both the engine and proxy components, including VPC and subnet creation, availability zone and flavor query, instance creation, and execution of the minor version upgrade task.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS Flavors (data.huaweicloud_dcs_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### Resources

- [VPC (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DCS Instance (huaweicloud_dcs_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS Instance Minor Version Upgrade (huaweicloud_dcs_instance_minor_version_upgrade)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance_minor_version_upgrade)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
            └── huaweicloud_dcs_instance_minor_version_upgrade

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
# Create a VPC resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

### 3. Create a VPC Subnet

Add the following script in the TF file (such as main.tf) to create a VPC subnet:

```hcl
# Create a VPC subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC resource created above
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet, assigned by referencing the input variable subnet_cidr; when this variable is empty, the subnet CIDR is automatically calculated based on the VPC CIDR
- **gateway_ip**: The gateway IP address of the subnet, assigned by referencing the input variable subnet_gateway_ip; when this variable is empty, the gateway IP is automatically calculated based on the subnet CIDR

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
- **count**: The number of data sources to create. When the input variable availability_zone is empty, this data source is created to automatically query the availability zones in the current region

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
  default     = 1
}

variable "instance_engine_version" {
  description = "The engine version of the Redis single instance"
  type        = string
  default     = "6.0"
}

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  engine         = "Redis"
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**Parameter Description**:
- **count**: The number of data sources to create. When the input variable instance_flavor_id is empty, this data source is created to automatically query the matching DCS flavors
- **engine**: The cache engine type, fixed to Redis
- **capacity**: The instance capacity (in GB), assigned by referencing the input variable instance_capacity
- **engine_version**: The cache engine version, assigned by referencing the input variable instance_engine_version

### 6. Create a Random Password

Add the following script in the TF file (such as main.tf) to create a random password:

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
- **count**: The number of resources to create. When the input variable instance_password is empty, this resource is created to automatically generate the instance password
- **length**: The length of the random password, set to 12
- **special**: Whether to include special characters, set to true
- **override_special**: The set of allowed special characters
- **min_upper**: The minimum number of uppercase letters
- **min_lower**: The minimum number of lowercase letters
- **min_numeric**: The minimum number of digits
- **min_special**: The minimum number of special characters

### 7. Create a DCS Instance

Add the following script in the TF file (such as main.tf) to create a DCS instance:

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
- **engine_version**: The cache engine version, assigned by referencing the input variable instance_engine_version
- **capacity**: The instance capacity (in GB), assigned by referencing the input variable instance_capacity
- **vpc_id**: The ID of the VPC to which the instance belongs, assigned by referencing the ID of the VPC resource created above
- **subnet_id**: The ID of the subnet to which the instance belongs, assigned by referencing the ID of the subnet resource created above
- **password**: The instance password. When the input variable instance_password is not empty, this variable is used; otherwise, the password generated by the random password resource is used
- **flavor**: The instance flavor. When the input variable instance_flavor_id is not empty, this variable is used; otherwise, the flavor name queried by the data source is used
- **availability_zones**: The list of availability zones to which the instance belongs. When the input variable availability_zone is not empty, this variable is used; otherwise, the first availability zone queried by the data source is used

### 8. Create a DCS Instance Minor Version Upgrade

Add the following script in the TF file (such as main.tf) to trigger the DCS instance minor version upgrade:

```hcl
# Create a DCS instance minor version upgrade resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "engine_minor_version" {
  description = "The target engine minor version. Use 'latest' for the latest version"
  type        = string
  default     = "latest"
}

variable "proxy_minor_version" {
  description = "The target proxy minor version. Use 'latest' for the latest version"
  type        = string
  default     = "latest"
}

resource "huaweicloud_dcs_instance_minor_version_upgrade" "test" {
  instance_id          = huaweicloud_dcs_instance.test.id
  proxy_minor_version  = var.proxy_minor_version
  engine_minor_version = var.engine_minor_version
}
```

**Parameter Description**:
- **instance_id**: The ID of the DCS instance to be upgraded, assigned by referencing the ID of the DCS instance resource created above
- **proxy_minor_version**: The target proxy minor version, assigned by referencing the input variable proxy_minor_version; the value `latest` means upgrading to the latest version
- **engine_minor_version**: The target engine minor version, assigned by referencing the input variable engine_minor_version; the value `latest` means upgrading to the latest version

> Note: This resource is an action resource. The create operation calls the DCS API to trigger the minor version upgrade, while the read, update, and delete operations are no-ops, and it does not support import. Changing any parameter will trigger resource replacement.

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
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

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DCS instance and triggering the minor version upgrade
4. Run `terraform show` to view the created DCS instance and minor version upgrade resource

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Instance Minor Version Upgrade](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-minor-version-upgrade)
