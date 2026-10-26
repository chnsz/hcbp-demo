# Deploy GeminiDB InfluxDB Instance

## Application Scenario

GeminiDB InfluxDB is a time series database service provided by Huawei Cloud, compatible with the InfluxDB 1.8 interface, and suitable for high-concurrency writes and queries of massive time series data in scenarios such as IoT, monitoring, and log analysis. This practice targets businesses that need to quickly build a high-performance time series data storage foundation, and uses Terraform to automate the creation of networks, security groups, instances, and backups.

This best practice will introduce how to use Terraform to automatically deploy a GeminiDB InfluxDB instance, including VPC and subnet creation, availability zone and flavor query, security group configuration, random password generation, instance creation, and instance backup.

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
    └── data.huaweicloud_gaussdb_nosql_flavors
        └── huaweicloud_geminidb_instance
            └── huaweicloud_geminidb_backup

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_geminidb_instance

huaweicloud_networking_secgroup
    └── huaweicloud_geminidb_instance

random_password
    └── huaweicloud_geminidb_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a VPC and Subnet

Add the following script in the TF file (such as main.tf):

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
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr, defaulting to 192.168.0.0/16
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC resource
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet. When the input variable subnet_cidr is empty, it is automatically divided based on the VPC CIDR; otherwise, it is assigned by referencing the input variable subnet_cidr
- **gateway_ip**: The gateway IP address of the subnet. When the input variable gateway_ip is empty, it is automatically calculated based on the subnet CIDR; otherwise, it is assigned by referencing the input variable gateway_ip

### 3. Query Availability Zones and GeminiDB Flavors

Add the following script in the TF file (such as main.tf):

```hcl
# Query availability zones and GeminiDB flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vcpus" {
  description = "The number of vCPUs of the flavor to query"
  type        = number
  default     = 4
}

variable "availability_zone" {
  description = "The availability zone of the GeminiDB InfluxDB instance. If not specified, the first AZ from data source will be used"
  type        = string
  default     = ""
}

data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_gaussdb_nosql_flavors" "test" {
  vcpus             = var.vcpus
  engine            = "influxdb"
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **vcpus**: The number of vCPUs of the flavor to query, assigned by referencing the input variable vcpus, defaulting to 4
- **engine**: The database engine type, fixed to influxdb
- **availability_zone**: The availability zone. When the input variable availability_zone is empty, the first availability zone in the availability zone list is used; otherwise, it is assigned by referencing the input variable availability_zone

### 4. Create a Security Group

Add the following script in the TF file (such as main.tf):

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
- **delete_default_rules**: Whether to delete the default rules of the security group, set to true

### 5. Generate a Random Password for the Instance

Add the following script in the TF file (such as main.tf):

```hcl
# Generate a random password for the instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_password" {
  description = "The password for the GeminiDB InfluxDB instance"
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
- **count**: Creates a random password when the input variable instance_password is empty; otherwise, it is not created
- **length**: The password length, set to 16
- **special**: Whether to include special characters, set to true
- **override_special**: The set of allowed special characters
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters in the password

### 6. Create a GeminiDB InfluxDB Instance

Add the following script in the TF file (such as main.tf):

```hcl
# Create a GeminiDB InfluxDB instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The GeminiDB InfluxDB instance name"
  type        = string
}

variable "instance_mode" {
  description = "The instance mode. Valid values are Cluster, EnhancedCluster, InfluxdbSingle"
  type        = string
  default     = "InfluxdbSingle"
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
  description = "The node quantity of the GeminiDB InfluxDB instance"
  type        = number
  default     = 1
}

variable "instance_flavor_size" {
  description = "The disk size in GB of the GeminiDB InfluxDB instance"
  type        = number
  default     = 100
}

variable "instance_flavor_storage" {
  description = "The disk type of the GeminiDB InfluxDB instance"
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

variable "cold_storage_size" {
  description = "The size of the cold storage in GB, 0 means no cold storage"
  type        = number
  default     = 0
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
  description = "The tags of the GeminiDB InfluxDB instance"
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
    type           = "influxdb"
    version        = "1.8"
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

  cold_storage_size = var.cold_storage_size

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
      bucket_name,
      data_export_switch,
      cold_storage_size,
      auto_renew,
      period,
      period_unit,
    ]
  }
}
```

**Parameter Description**:
- **name**: The instance name, assigned by referencing the input variable instance_name
- **availability_zone**: The availability zone. When the input variable availability_zone is empty, the first availability zone in the availability zone list is used; otherwise, it is assigned by referencing the input variable availability_zone
- **vpc_id**: The ID of the VPC to which the instance belongs, assigned by referencing the ID of the VPC resource
- **subnet_id**: The ID of the subnet to which the instance belongs, assigned by referencing the ID of the subnet resource
- **security_group_id**: The ID of the security group to which the instance belongs, assigned by referencing the ID of the security group resource
- **password**: The instance password. When the input variable instance_password is not empty, this value is used; otherwise, the result generated by the random password resource is used
- **mode**: The instance mode, assigned by referencing the input variable instance_mode, defaulting to InfluxdbSingle
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, defaulting to 0
- **ssl_option**: Whether to enable SSL, assigned by referencing the input variable instance_ssl_option, defaulting to on
- **maintenance_start_time**, **maintenance_end_time**: The start and end time of the maintenance window, assigned by referencing the input variables maintenance_start_time and maintenance_end_time respectively
- **datastore.type**: The database engine type, fixed to influxdb
- **datastore.version**: The database engine version, fixed to 1.8
- **datastore.storage_engine**: The storage engine, fixed to rocksDB
- **flavor.num**: The number of nodes, assigned by referencing the input variable instance_flavor_num
- **flavor.size**: The disk size in GB, assigned by referencing the input variable instance_flavor_size
- **flavor.storage**: The disk type, assigned by referencing the input variable instance_flavor_storage
- **flavor.spec_code**: The resource specification code. When the input variable instance_flavor_spec_code is not empty, this value is used; otherwise, the first flavor name returned by the flavor query data source is used
- **backup_strategy.start_time**: The backup time window, assigned by referencing the input variable instance_backup_time_window
- **backup_strategy.keep_days**: The number of days to retain backups, assigned by referencing the input variable instance_backup_keep_days
- **cold_storage_size**: The size of the cold storage in GB, assigned by referencing the input variable cold_storage_size, where 0 means cold storage is not enabled
- **charging_mode**: The charging mode, assigned by referencing the input variable charging_mode, defaulting to prePaid
- **period_unit**: The charging period unit, assigned by referencing the input variable period_unit, defaulting to month
- **auto_renew**: Whether to enable auto-renew, assigned by referencing the input variable auto_renew, defaulting to true
- **period**: The charging period, assigned by referencing the input variable period, defaulting to 1
- **tags**: The instance tags, assigned by referencing the input variable tags
- **lifecycle.ignore_changes**: Ignores attributes that may drift after creation, including password, flavor.0.storage, ssl_option, bucket_name, data_export_switch, cold_storage_size, auto_renew, period, and period_unit

### 7. Create a GeminiDB Backup

Add the following script in the TF file (such as main.tf):

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
- **depends_on**: Explicitly declares the dependency to ensure the backup is created after the instance is created

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
vpc_name                    = "your_vpc1"
subnet_name                 = "your_subnet"
security_group_name         = "your_security_group"
instance_name               = "your_geminidb_influxdb_instance"
instance_backup_time_window = "03:00-04:00"
instance_backup_keep_days   = 14
tags                        = {
  foo = "bar"
  key = "value"
}
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GeminiDB InfluxDB instance
4. Run `terraform show` to view the created GeminiDB InfluxDB instance

## Reference Information

- [Huawei Cloud GeminiDB Product Documentation](https://support.huaweicloud.com/geminidb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GeminiDB InfluxDB Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/geminidb/geminidb-influxdb-instance)
