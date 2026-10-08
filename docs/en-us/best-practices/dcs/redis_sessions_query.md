# Deploy Redis Client Sessions Query

## Application Scenario

During operation, a DCS Redis instance establishes a large number of connections with clients. O&M personnel need to view the active client sessions on the instance to locate abnormal connections, troubleshoot performance issues, or perform security audits.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis single instance and query the client sessions of the instance based on the instance shard information, including VPC creation, subnet configuration, availability zone and flavor queries, instance configuration, instance shard query, and session query.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS Flavors (data.huaweicloud_dcs_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)
- [DCS Instance Shards (data.huaweicloud_dcs_instance_shards)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_instance_shards)

### Resources

- [VPC (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DCS Instance (huaweicloud_dcs_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS Sessions Query (huaweicloud_dcs_sessions_query)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_sessions_query)

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
                └── huaweicloud_dcs_sessions_query

random_password
    └── huaweicloud_dcs_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a VPC

Add the following script to the TF file (such as main.tf) to create a VPC:

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
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr

### 3. Create a Subnet

Add the following script to the TF file to create a subnet:

```hcl
# Create a subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **cidr**: The subnet CIDR block; when the input variable subnet_cidr is empty, the subnet CIDR block is automatically divided based on the VPC CIDR block
- **gateway_ip**: The subnet gateway IP; when the input variable subnet_gateway_ip is empty, the gateway IP is automatically calculated based on the subnet CIDR block

### 4. Query the Availability Zones

Add the following script to the TF file to query the availability zones:

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
- **count**: The data source is queried only when the input variable availability_zone is empty, to automatically obtain the availability zone

### 5. Query the DCS Flavors

Add the following script to the TF file to query the DCS flavors:

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
  default     = 4
}

variable "instance_engine_version" {
  description = "The engine version of the Redis single instance"
  type        = string
  default     = "5.0"
}

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  engine         = "Redis"
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**Parameter Description**:
- **count**: The data source is queried only when the input variable instance_flavor_id is empty, to automatically obtain the flavor
- **engine**: The cache engine type, fixed to Redis
- **capacity**: The instance capacity, assigned by referencing the input variable instance_capacity
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version

### 6. Create a Random Password

Add the following script to the TF file to create a random password:

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
- **count**: The random password resource is created only when the input variable instance_password is empty
- **length**: The password length, set to 12
- **special**: Whether to include special characters, set to true
- **override_special**: The set of allowed special characters
- **min_upper**: The minimum number of uppercase letters
- **min_lower**: The minimum number of lowercase letters
- **min_numeric**: The minimum number of digits
- **min_special**: The minimum number of special characters

### 7. Create a DCS Redis Instance

Add the following script to the TF file to create a DCS Redis instance:

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
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC resource created in the previous step
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet resource created in the previous step
- **password**: The instance password; when the input variable instance_password is not empty, this value is used; otherwise, the result generated by the random password resource is used
- **flavor**: The instance flavor; when the input variable instance_flavor_id is not empty, this value is used; otherwise, the first flavor name queried by the flavors data source is used
- **availability_zones**: The list of availability zones to which the instance belongs; when the input variable availability_zone is not empty, this value is used; otherwise, the first availability zone queried by the availability zones data source is used

### 8. Query the DCS Instance Shards

Add the following script to the TF file to query the DCS instance shards:

```hcl
# Query the DCS instance shards in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_dcs_instance_shards" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
}

locals {
  replication_list = try(data.huaweicloud_dcs_instance_shards.test.group_list[0].replication_list, [])
}
```

**Parameter Description**:
- **instance_id**: The instance ID, referencing the ID of the DCS instance resource created in the previous step
- **replication_list**: A local variable used to extract the replication list from the shards data source for subsequent session query

### 9. Query the DCS Client Sessions

Add the following script to the TF file to query the DCS client sessions:

```hcl
# Query the DCS client sessions in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "clean_cache" {
  description = "Whether to re-query and save the session list"
  type        = bool
  default     = false
}

resource "huaweicloud_dcs_sessions_query" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
  node_id     = try(local.replication_list[0].node_id, "")
  clean_cache = var.clean_cache
}
```

**Parameter Description**:
- **instance_id**: The instance ID, referencing the ID of the DCS instance resource created in the previous step
- **node_id**: The node ID, automatically obtained from the replication list of the instance shards data source
- **clean_cache**: Whether to re-query and save the session list, assigned by referencing the input variable clean_cache

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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DCS Redis instance and querying the client sessions
4. Run `terraform show` to view the created DCS Redis instance and the queried client sessions

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Client Sessions Query](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-sessions-query)
