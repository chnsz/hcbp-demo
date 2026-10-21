# Deploy GeminiDB DynamoDB-Compatible Instance

## Application Scenario

GeminiDB is a cloud-native multi-model NoSQL database service provided by Huawei Cloud, compatible with multiple mainstream NoSQL interfaces such as Cassandra, MongoDB, Redis, and InfluxDB. The DynamoDB-compatible instance is built on the RocksDB storage engine and exposes a DynamoDB-compatible interface, making it suitable for key-value NoSQL scenarios that require a managed database with high availability and elastic scalability.

This best practice will introduce how to use Terraform to automatically deploy a GeminiDB DynamoDB-compatible instance, including VPC and subnet creation, availability zone and flavor query, security group configuration, instance creation, and instance manual backup.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [GeminiDB Flavors (data.huaweicloud_gaussdb_nosql_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_nosql_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GeminiDB Instance (huaweicloud_geminidb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/geminidb_instance)
- [GeminiDB Backup (huaweicloud_geminidb_backup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/geminidb_backup)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    ├── data.huaweicloud_gaussdb_nosql_flavors.test
    └── huaweicloud_geminidb_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
            └── huaweicloud_geminidb_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_geminidb_instance.test

random_password.test
    └── huaweicloud_geminidb_instance.test

huaweicloud_geminidb_instance.test
    └── huaweicloud_geminidb_backup.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a VPC and Subnet

Add the following script in the TF file (such as main.tf):

```hcl
# Create a VPC and subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

variable "subnet_name" {
  description = "The name of the VPC subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the VPC subnet, defaults to a /24 derived from the VPC CIDR"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the VPC subnet, defaults to the first IP of the derived subnet CIDR"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr
- **vpc_id**: The VPC to which the subnet belongs, referencing the ID of the created VPC
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The subnet CIDR block; when the input variable subnet_cidr is empty, a /24 subnet is derived from the VPC CIDR
- **gateway_ip**: The subnet gateway IP; when the input variable subnet_gateway_ip is empty, the first IP of the derived subnet is used

### 3. Query Availability Zones and GeminiDB Flavors

Add the following script in the TF file (such as main.tf):

```hcl
# Query availability zones and GeminiDB flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vcpus" {
  description = "The number of vCPUs of the flavor to query"
  type        = number
  default     = 2
}

variable "availability_zone" {
  description = "The availability zone of the GeminiDB DynamoDB instance, uses the first AZ from data source if empty"
  type        = string
  default     = ""
}

data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_gaussdb_nosql_flavors" "test" {
  vcpus             = var.vcpus
  engine            = "cassandra"
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **vcpus**: The number of vCPUs of the flavor to query, assigned by referencing the input variable vcpus
- **engine**: The engine type of the flavor to query; the DynamoDB-compatible instance shares the underlying Cassandra infrastructure, so it is fixed to cassandra
- **availability_zone**: The availability zone of the flavor; when the input variable availability_zone is empty, the first availability zone in the list is used

### 4. Create a Security Group

Add the following script in the TF file (such as main.tf):

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group; set to true so that access rules can be customized as needed

### 5. Generate a Random Password

Add the following script in the TF file (such as main.tf):

```hcl
# Generate a random password when no instance password is provided
variable "instance_password" {
  description = "The password of the GeminiDB DynamoDB instance, generated randomly if left empty"
  type        = string
  sensitive   = true
  default     = ""
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
- **count**: Creates a random password when the input variable instance_password is empty; otherwise, it is not created
- **length**: The length of the random password, set to 12
- **special**: Whether to include special characters, set to true
- **override_special**: The set of allowed special characters
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters required respectively

### 6. Create a GeminiDB DynamoDB-Compatible Instance

Add the following script in the TF file (such as main.tf):

```hcl
# Create a GeminiDB DynamoDB-compatible instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the GeminiDB DynamoDB instance"
  type        = string
}

variable "instance_mode" {
  description = "The instance type of the GeminiDB DynamoDB instance"
  type        = string
  default     = "Cluster"
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the GeminiDB DynamoDB instance"
  type        = string
  default     = "0"
}

variable "instance_ssl_option" {
  description = "Whether SSL is enabled for the GeminiDB DynamoDB instance"
  type        = string
  default     = "on"
}

variable "instance_flavor_num" {
  description = "The node quantity of the GeminiDB DynamoDB instance"
  type        = number
  default     = 3
}

variable "instance_flavor_size" {
  description = "The disk size (GB) of the GeminiDB DynamoDB instance"
  type        = number
  default     = 200
}

variable "instance_flavor_storage" {
  description = "The disk type of the GeminiDB DynamoDB instance"
  type        = string
  default     = "ULTRAHIGH"
}

variable "instance_flavor_spec_code" {
  description = "The resource specification code, uses the first flavor from data source if empty"
  type        = string
  default     = ""
}

variable "instance_backup_time_window" {
  description = "The backup time window of the GeminiDB DynamoDB instance"
  type        = string
  default     = "03:00-04:00"
}

variable "instance_backup_keep_days" {
  description = "The number of days to retain backup files of the GeminiDB DynamoDB instance"
  type        = number
  default     = 14
}

variable "charging_mode" {
  description = "The charging mode of the GeminiDB DynamoDB instance, postPaid or prePaid"
  type        = string
  default     = "prePaid"
}

variable "period_unit" {
  description = "The charging period unit of the GeminiDB DynamoDB instance, month or year"
  type        = string
  default     = "month"
}

variable "auto_renew" {
  description = "Whether auto-renew is enabled for the GeminiDB DynamoDB instance"
  type        = string
  default     = "true"
}

variable "period" {
  description = "The charging period of the GeminiDB DynamoDB instance"
  type        = number
  default     = 1
}

variable "tags" {
  description = "The key/value pairs to associate with the GeminiDB DynamoDB instance"
  type        = map(string)
  default     = {
    foo = "bar"
    key = "value"
  }
}

resource "huaweicloud_geminidb_instance" "test" {
  name                  = var.instance_name
  availability_zone     = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  password              = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  mode                  = var.instance_mode
  enterprise_project_id = var.enterprise_project_id
  ssl_option            = var.instance_ssl_option

  datastore {
    type           = "dynamodb"
    version        = ""
    storage_engine = "rocksDB"
  }

  flavor {
    num       = var.instance_flavor_num
    size      = var.instance_flavor_size
    storage   = var.instance_flavor_storage
    spec_code = var.instance_flavor_spec_code != "" ? var.instance_flavor_spec_code : try(data.huaweicloud_gaussdb_nosql_flavors.test.flavors[0].name, null)
  }

  backup_strategy {
    start_time = var.instance_backup_time_window
    keep_days  = var.instance_backup_keep_days
  }

  charging_mode = var.charging_mode
  period_unit   = var.period_unit
  auto_renew    = var.auto_renew
  period        = var.period

  tags = var.tags

  lifecycle {
    ignore_changes = [
      password,
      flavor.0.storage,
      ssl_option,
      auto_renew,
      period,
      period_unit,
    ]
  }
}
```

**Parameter Description**:
- **name**: The instance name, assigned by referencing the input variable instance_name
- **availability_zone**: The instance availability zone; when the input variable availability_zone is empty, the first availability zone in the list is used
- **vpc_id**: The VPC to which the instance belongs, referencing the ID of the created VPC
- **subnet_id**: The subnet to which the instance belongs, referencing the ID of the created subnet
- **security_group_id**: The security group to which the instance belongs, referencing the ID of the created security group
- **password**: The instance password; when the input variable instance_password is not empty, this value is used; otherwise, the randomly generated password is used
- **mode**: The instance type; a DynamoDB-compatible instance must be Cluster, assigned by referencing the input variable instance_mode
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **ssl_option**: Whether SSL is enabled, assigned by referencing the input variable instance_ssl_option
- **datastore.type**: The database type, fixed to dynamodb
- **datastore.version**: The database version, an empty string for the DynamoDB-compatible instance
- **datastore.storage_engine**: The storage engine, fixed to rocksDB
- **flavor.num**: The number of nodes, assigned by referencing the input variable instance_flavor_num
- **flavor.size**: The disk size (GB), assigned by referencing the input variable instance_flavor_size
- **flavor.storage**: The disk type, assigned by referencing the input variable instance_flavor_storage
- **flavor.spec_code**: The resource specification code; when the input variable instance_flavor_spec_code is empty, the first flavor in the flavor list is used
- **backup_strategy.start_time**: The backup time window, assigned by referencing the input variable instance_backup_time_window
- **backup_strategy.keep_days**: The number of days to retain backups, assigned by referencing the input variable instance_backup_keep_days
- **charging_mode**: The charging mode, assigned by referencing the input variable charging_mode
- **period_unit**: The charging period unit, assigned by referencing the input variable period_unit
- **auto_renew**: Whether auto-renew is enabled, assigned by referencing the input variable auto_renew
- **period**: The charging period, assigned by referencing the input variable period
- **tags**: The instance tags, assigned by referencing the input variable tags
- **lifecycle.ignore_changes**: Ignores attributes that may drift after creation, including password, flavor.0.storage, ssl_option, auto_renew, period, and period_unit

### 7. Create a GeminiDB Backup

Add the following script in the TF file (such as main.tf):

```hcl
# Create a GeminiDB backup in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "backup_name" {
  description = "The name of the GeminiDB backup"
  type        = string
  default     = "tf_test_backup"
}

variable "backup_description" {
  description = "The description of the GeminiDB backup"
  type        = string
  default     = "Created by Terraform"
}

resource "huaweicloud_geminidb_backup" "test" {
  instance_id = huaweicloud_geminidb_instance.test.id
  name        = var.backup_name
  description = var.backup_description

  depends_on = [huaweicloud_geminidb_instance.test]
}
```

**Parameter Description**:
- **instance_id**: The instance to which the backup belongs, referencing the ID of the created GeminiDB instance
- **name**: The backup name, assigned by referencing the input variable backup_name
- **description**: The backup description, assigned by referencing the input variable backup_description
- **depends_on**: Explicitly declares the dependency to ensure the backup is created after the instance is created

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# VPC and subnet variables
vpc_name            = "tf_test_vpc"
vpc_cidr            = "192.168.0.0/16"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"

# GeminiDB DynamoDB instance variables
instance_name = "tf_test_geminidb_dynamodb_instance"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GeminiDB DynamoDB-compatible instance
4. Run `terraform show` to view the created GeminiDB DynamoDB-compatible instance

## Reference Information

- [Huawei Cloud GeminiDB Product Documentation](https://support.huaweicloud.com/geminidb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GeminiDB DynamoDB-Compatible Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/geminidb/geminidb-dynamodb-instance)
