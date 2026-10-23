# Deploy GeminiDB HBase Instance

## Application Scenario

GeminiDB is a cloud-native multi-model NoSQL database service provided by Huawei Cloud, compatible with multiple mainstream NoSQL interfaces such as HBase, Cassandra, MongoDB, Redis, and InfluxDB. The GeminiDB HBase-compatible instance is built on the RocksDB storage engine and is suitable for high-concurrency read and write of massive data, wide-table storage, and real-time query scenarios, helping enterprises build a stable NoSQL data foundation with lower O&M costs.

This best practice will introduce how to use Terraform to automatically deploy a GeminiDB HBase instance, including VPC and subnet creation, availability zone and flavor query, security group configuration, random password generation, instance creation, and instance backup.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [GeminiDB Flavors (data.huaweicloud_gaussdb_nosql_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_nosql_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GeminiDB Instance (huaweicloud_geminidb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/geminidb_instance)
- [GeminiDB Backup (huaweicloud_geminidb_backup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/geminidb_backup)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_gaussdb_nosql_flavors
    └── huaweicloud_geminidb_instance

data.huaweicloud_gaussdb_nosql_flavors
    └── huaweicloud_geminidb_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_geminidb_instance

huaweicloud_networking_secgroup
    └── huaweicloud_geminidb_instance

random_password
    └── huaweicloud_geminidb_instance

huaweicloud_geminidb_instance
    └── huaweicloud_geminidb_backup
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a VPC and Subnet

Add the following script to the TF file (such as main.tf) to create a VPC and subnet:

```hcl
# Create VPC and subnet resources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
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
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr, defaulting to `192.168.0.0/16`
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC resource
- **cidr**: The CIDR block of the subnet, automatically divided from the VPC CIDR block via `cidrsubnet` when not specified
- **gateway_ip**: The gateway IP address of the subnet, automatically calculated from the subnet CIDR block via `cidrhost` when not specified

### 3. Query Availability Zones and GeminiDB Flavors

Add the following script to the TF file (such as main.tf) to query the availability zone list and GeminiDB flavors:

```hcl
# Query availability zones and GeminiDB flavors data sources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vcpus" {
  description = "The number of vCPUs of the flavor to query"
  type        = number
  default     = 2
}

variable "availability_zone" {
  description = "The availability zone of the GeminiDB HBase instance. If not specified, the first AZ from data source will be used"
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
- **vcpus**: The number of vCPUs of the flavor to query, assigned by referencing the input variable vcpus, defaulting to `2`
- **engine**: The engine type of the flavor to query, fixed to `cassandra` (the GeminiDB HBase instance reuses this engine parameter for flavor query)
- **availability_zone**: The availability zone, assigned by referencing the input variable availability_zone, using the first availability zone from the list when not specified

### 4. Create a Security Group

Add the following script to the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group, set to `true`

### 5. Generate the Instance Password

Add the following script to the TF file (such as main.tf) to automatically generate a random password when no password is specified:

```hcl
# Generate a random password resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_password" {
  description = "The password for the GeminiDB HBase instance"
  type        = string
  default     = ""
  sensitive   = true
}

resource "random_password" "test" {
  count = var.instance_password == "" ? 1 : 0

  length           = 16
  special          = true
  override_special = "!@%^*-_=+"
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**Parameter Description**:
- **count**: Creates the random password when the input variable instance_password is empty, otherwise not
- **length**: The password length, set to `16`
- **special**: Whether to include special characters, set to `true`
- **override_special**: The set of allowed special characters
- **min_upper**/**min_lower**/**min_numeric**/**min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters

### 6. Create a GeminiDB HBase Instance

Add the following script to the TF file (such as main.tf) to create a GeminiDB HBase instance:

```hcl
# Create a GeminiDB instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The GeminiDB HBase instance name"
  type        = string
}

variable "instance_mode" {
  description = "The instance mode. Valid value is Cluster"
  type        = string
  default     = "Cluster"
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = "0"
}

variable "instance_ssl_option" {
  description = "Whether to enable SSL. Valid values are on, off"
  type        = string
  default     = "on"
}

variable "maintenance_start_time" {
  description = "The start time of the maintenance window in UTC HH:MM format"
  type        = string
  default     = "02:00"
}

variable "maintenance_end_time" {
  description = "The end time of the maintenance window in UTC HH:MM format, 4 hours after start time"
  type        = string
  default     = "06:00"
}

variable "instance_flavor_num" {
  description = "The node quantity of the GeminiDB HBase instance"
  type        = number
  default     = 3
}

variable "instance_flavor_size" {
  description = "The disk size in GB of the GeminiDB HBase instance"
  type        = number
  default     = 200
}

variable "instance_flavor_storage" {
  description = "The disk type of the GeminiDB HBase instance"
  type        = string
  default     = "ULTRAHIGH"
}

variable "instance_flavor_spec_code" {
  description = "The resource specification code. If not specified, the first available flavor will be used"
  type        = string
  default     = ""
}

variable "instance_backup_time_window" {
  description = "The backup time window in HH:MM-HH:MM format"
  type        = string
}

variable "instance_backup_keep_days" {
  description = "The number of days to retain backups"
  type        = number
}

variable "charging_mode" {
  description = "The charging mode. Valid values are prePaid, postPaid"
  type        = string
  default     = "prePaid"
}

variable "period_unit" {
  description = "The charging period unit. Valid values are month, year"
  type        = string
  default     = "month"
}

variable "auto_renew" {
  description = "Whether to enable auto-renew. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "period" {
  description = "The charging period"
  type        = number
  default     = 1
}

variable "tags" {
  description = "The tags of the GeminiDB HBase instance"
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

  maintenance_start_time = var.maintenance_start_time
  maintenance_end_time   = var.maintenance_end_time

  datastore {
    type           = "hbase"
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
- **availability_zone**: The instance availability zone, assigned by referencing the input variable availability_zone, using the first availability zone from the list when not specified
- **vpc_id**/**subnet_id**/**security_group_id**: The VPC, subnet, and security group to which the instance belongs, assigned by referencing the IDs of the corresponding resources
- **password**: The instance password, assigned by referencing the input variable instance_password, using the result generated by the random password resource when empty
- **mode**: The instance mode, assigned by referencing the input variable instance_mode, which must be `Cluster` for a GeminiDB HBase instance
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **ssl_option**: Whether to enable SSL, assigned by referencing the input variable instance_ssl_option
- **maintenance_start_time**/**maintenance_end_time**: The start and end time of the maintenance window, assigned by referencing the input variables maintenance_start_time and maintenance_end_time respectively
- **datastore.type**: The data store type, fixed to `hbase`
- **datastore.version**: The data store version, using an empty string for HBase instances
- **datastore.storage_engine**: The storage engine, fixed to `rocksDB`
- **flavor.num**: The node quantity, assigned by referencing the input variable instance_flavor_num
- **flavor.size**: The disk size in GB, assigned by referencing the input variable instance_flavor_size
- **flavor.storage**: The disk type, assigned by referencing the input variable instance_flavor_storage
- **flavor.spec_code**: The resource specification code, assigned by referencing the input variable instance_flavor_spec_code, using the first flavor from the flavor list when empty
- **backup_strategy.start_time**/**backup_strategy.keep_days**: The backup time window and backup retention days, assigned by referencing the input variables instance_backup_time_window and instance_backup_keep_days respectively
- **charging_mode**/**period_unit**/**auto_renew**/**period**: The charging mode, charging period unit, auto-renew, and charging period, assigned by referencing the input variables charging_mode, period_unit, auto_renew, and period respectively
- **tags**: The instance tags, assigned by referencing the input variable tags
- **lifecycle.ignore_changes**: Ignores attributes that may drift after creation, including password, flavor.0.storage, ssl_option, auto_renew, period, and period_unit

### 7. Create a GeminiDB Backup

Add the following script to the TF file (such as main.tf) to create a GeminiDB backup:

```hcl
# Create a GeminiDB backup resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **instance_id**: The ID of the instance to which the backup belongs, assigned by referencing the ID of the GeminiDB instance resource
- **name**: The backup name, assigned by referencing the input variable backup_name
- **description**: The backup description, assigned by referencing the input variable backup_description

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
vpc_name                    = "your_vpc1"
subnet_name                 = "your_subnet"
security_group_name         = "your_security_group"
instance_name               = "your_geminidb_hbase_instance"
instance_backup_time_window = "03:00-04:00"
instance_backup_keep_days   = 14
tags                        = {
  foo = "bar"
  key = "value"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GeminiDB HBase instance
4. Run `terraform show` to view the created GeminiDB HBase instance

## Reference Information

- [Huawei Cloud GeminiDB Product Documentation](https://support.huaweicloud.com/geminidb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GeminiDB HBase Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/geminidb/geminidb-hbase-instance)
