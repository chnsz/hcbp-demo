# Deploy TaurusDB Instance with LTS Log

## Application Scenario

TaurusDB is an enterprise-grade cloud-native database service provided by Huawei Cloud, fully compatible with the MySQL protocol and built on a compute-storage separation architecture that supports one-writer-multiple-readers, second-level elastic scaling, and massive data storage. In production environments, database error logs and slow query logs are essential for troubleshooting and performance analysis; if they are kept only on the instance, they are neither easy to retain for the long term nor convenient to search and analyze centrally.

This best practice will introduce how to use Terraform to automatically deploy a TaurusDB instance, create a Log Tank Service (LTS) log group and log stream, and associate the instance logs with the LTS log stream through the TaurusDB LTS log capability, so that logs are centrally stored, searched, and managed throughout their lifecycle.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [TaurusDB Flavors (data.huaweicloud_taurusdb_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [TaurusDB Instance (huaweicloud_taurusdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_instance)
- [LTS Log Group (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS Log Stream (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [TaurusDB LTS Log (huaweicloud_taurusdb_lts_log)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_lts_log)

### Resource/Data Source Dependencies

```
data.huaweicloud_taurusdb_flavors.test
    ├── huaweicloud_taurusdb_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
            └── huaweicloud_taurusdb_instance.test

huaweicloud_networking_secgroup.test
    ├── huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_taurusdb_instance.test

random_password.test
    └── huaweicloud_taurusdb_instance.test

huaweicloud_lts_group.test
    ├── huaweicloud_lts_stream.test
    └── huaweicloud_taurusdb_lts_log.test

huaweicloud_lts_stream.test
    └── huaweicloud_taurusdb_lts_log.test

huaweicloud_taurusdb_instance.test
    └── huaweicloud_taurusdb_lts_log.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a VPC and Subnet

Add the following script to the TF file (such as main.tf):

```hcl
# Create a VPC and subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC created above
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The subnet CIDR block; when the input variable subnet_cidr is empty, it is automatically divided from the VPC CIDR block using the `cidrsubnet` function
- **gateway_ip**: The subnet gateway IP; when the input variable gateway_ip is empty, it is automatically calculated using the `cidrhost` function

### 3. Query TaurusDB Flavors

Add the following script to the TF file (such as main.tf):

```hcl
# Query TaurusDB flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone_mode" {
  description = "The availability zone mode. Valid values are single, multi"
  type        = string
  default     = "multi"
}

variable "master_availability_zone" {
  description = "The master availability zone of the TaurusDB instance. If not specified, the first available AZ from flavors will be used"
  type        = string
  default     = ""
}

data "huaweicloud_taurusdb_flavors" "test" {
  engine                 = "gaussdb-mysql"
  version                = "8.0"
  availability_zone_mode = var.availability_zone_mode
}

locals {
  # Obtain available AZs from the flavor's az_status (TaurusDB AZs must come from the flavor query result)
  available_azs = try([for k, v in data.huaweicloud_taurusdb_flavors.test.flavors[0].az_status : k if v == "normal"], [])
  master_az     = var.master_availability_zone != "" ? var.master_availability_zone : try(local.available_azs[0], "")
}
```

**Parameter Description**:
- **engine**: The database engine, fixed to `gaussdb-mysql`
- **version**: The database engine version, fixed to `8.0`
- **availability_zone_mode**: The availability zone mode, assigned by referencing the input variable availability_zone_mode
- **available_azs**: A local variable that filters the AZs whose status is `normal` from the `az_status` of the flavor query result
- **master_az**: A local variable that uses the first available AZ as the master AZ when the input variable master_availability_zone is empty

### 4. Create a Security Group and Access Rule

Add the following script to the TF file (such as main.tf):

```hcl
# Create a security group and its access rule in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

variable "instance_db_port" {
  description = "The database port"
  type        = number
  default     = 3306
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = var.vpc_cidr
  ports             = var.instance_db_port
  protocol          = "tcp"
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group; set to `true` so that only explicitly defined rules are retained
- **security_group_id**: The ID of the security group to which the rule belongs, assigned by referencing the ID of the security group created above
- **direction**: The rule direction, set to `ingress`
- **remote_ip_prefix**: The source CIDR block allowed to access, assigned by referencing the input variable vpc_cidr
- **ports**: The port allowed to access, assigned by referencing the input variable instance_db_port
- **protocol**: The protocol type, set to `tcp`

### 5. Create a Random Password

Add the following script to the TF file (such as main.tf):

```hcl
# Create a random password in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_password" {
  description = "The password for the TaurusDB instance"
  type        = string
  default     = ""
  sensitive   = true
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
- **count**: The random password is created when the input variable instance_password is empty; otherwise it is not created
- **length**: The password length, set to `12`
- **special**: Whether to include special characters, set to `true`
- **override_special**: The set of allowed special characters
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters respectively

### 6. Create a TaurusDB Instance

Add the following script to the TF file (such as main.tf):

```hcl
# Create a TaurusDB instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The TaurusDB instance name"
  type        = string
}

variable "instance_flavor_ref" {
  description = "The flavor code of the TaurusDB instance. If not specified, the first available flavor will be used"
  type        = string
  default     = ""
}

variable "instance_mode" {
  description = "The instance mode. Valid values are Cluster, StandSingle"
  type        = string
  default     = "Cluster"
}

variable "read_replicas" {
  description = "The number of read replicas"
  type        = number
  default     = 2
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = "0"
}

variable "volume_type" {
  description = "The storage type of the instance. Valid values are DL6, DL5"
  type        = string
  default     = "DL6"
}

variable "time_zone" {
  description = "The time zone of the instance"
  type        = string
  default     = "UTC+08:00"
}

variable "ssl_option" {
  description = "Whether to enable SSL. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "sql_filter_enabled" {
  description = "Whether to enable SQL filter"
  type        = bool
  default     = true
}

variable "slow_log_show_original_switch" {
  description = "Whether to enable slow log show original switch"
  type        = bool
  default     = true
}

variable "table_name_case_sensitivity" {
  description = "Whether the kernel table name is case sensitive"
  type        = bool
  default     = true
}

variable "multi_tenant_switch" {
  description = "Whether to enable multi-tenancy switch. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "maintain_begin" {
  description = "The start time of the maintenance window in HH:MM format"
  type        = string
  default     = "02:00"
}

variable "maintain_end" {
  description = "The end time of the maintenance window in HH:MM format"
  type        = string
  default     = "06:00"
}

variable "description" {
  description = "The description of the TaurusDB instance"
  type        = string
  default     = ""
}

variable "seconds_level_monitoring_enabled" {
  description = "Whether to enable seconds level monitoring"
  type        = bool
  default     = true
}

variable "seconds_level_monitoring_period" {
  description = "The seconds level collection period. Valid values are 1, 5"
  type        = number
  default     = 5
}

variable "audit_log_enabled" {
  description = "Whether to enable audit log"
  type        = bool
  default     = true
}

variable "audit_log_keep_days" {
  description = "The number of days for storing audit logs"
  type        = number
  default     = 7
}

variable "reserve_audit_logs" {
  description = "Whether to reserve historical audit logs when SQL audit is disabled. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "instance_backup_time_window" {
  description = "The backup time window in HH:MM-HH:MM format"
  type        = string
}

variable "instance_backup_keep_days" {
  description = "The number of days to retain backups"
  type        = number
}

variable "tags" {
  description = "The tags of the TaurusDB instance"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_taurusdb_instance" "test" {
  name                             = var.instance_name
  password                         = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  flavor                           = var.instance_flavor_ref != "" ? var.instance_flavor_ref : try(data.huaweicloud_taurusdb_flavors.test.flavors[0].name, "")
  vpc_id                           = huaweicloud_vpc.test.id
  subnet_id                        = huaweicloud_vpc_subnet.test.id
  security_group_id                = huaweicloud_networking_secgroup.test.id
  mode                             = var.instance_mode
  availability_zone_mode           = var.availability_zone_mode
  master_availability_zone         = local.master_az
  read_replicas                    = var.read_replicas
  enterprise_project_id            = var.enterprise_project_id
  volume_type                      = var.volume_type
  time_zone                        = var.time_zone
  port                             = var.instance_db_port
  ssl_option                       = var.ssl_option
  sql_filter_enabled               = var.sql_filter_enabled
  slow_log_show_original_switch    = var.slow_log_show_original_switch
  table_name_case_sensitivity      = var.table_name_case_sensitivity
  multi_tenant_switch              = var.multi_tenant_switch
  maintain_begin                   = var.maintain_begin
  maintain_end                     = var.maintain_end
  description                      = var.description
  seconds_level_monitoring_enabled = var.seconds_level_monitoring_enabled
  seconds_level_monitoring_period  = var.seconds_level_monitoring_enabled ? var.seconds_level_monitoring_period : null
  audit_log_enabled                = var.audit_log_enabled
  audit_log_keep_days              = var.audit_log_keep_days
  reserve_audit_logs               = var.reserve_audit_logs

  datastore {
    engine  = "gaussdb-mysql"
    version = "8.0"
  }

  backup_strategy {
    start_time = var.instance_backup_time_window
    keep_days  = tostring(var.instance_backup_keep_days)
  }

  tags = var.tags

  lifecycle {
    ignore_changes = [
      password, ssl_option, reserve_audit_logs, datastore[0].version,
    ]
  }
}
```

**Parameter Description**:
- **name**: The instance name, assigned by referencing the input variable instance_name
- **password**: The instance password; when the input variable instance_password is not empty, this value is used; otherwise the result generated by the random password resource is used
- **flavor**: The instance flavor; when the input variable instance_flavor_ref is not empty, this value is used; otherwise the first flavor in the flavor query result is used
- **vpc_id**, **subnet_id**, **security_group_id**: Assigned by referencing the IDs of the VPC, subnet, and security group created above respectively
- **mode**: The instance mode, assigned by referencing the input variable instance_mode
- **availability_zone_mode**, **master_availability_zone**: The availability zone mode and master AZ, assigned by referencing the input variable availability_zone_mode and the local variable master_az respectively
- **read_replicas**: The number of read replicas, assigned by referencing the input variable read_replicas
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **volume_type**: The storage type, assigned by referencing the input variable volume_type
- **time_zone**: The time zone, assigned by referencing the input variable time_zone
- **port**: The database port, assigned by referencing the input variable instance_db_port
- **ssl_option**: Whether to enable SSL, assigned by referencing the input variable ssl_option
- **sql_filter_enabled**, **slow_log_show_original_switch**, **table_name_case_sensitivity**, **multi_tenant_switch**: Assigned by referencing the corresponding input variables respectively
- **maintain_begin**, **maintain_end**: The start and end time of the maintenance window, assigned by referencing the input variables maintain_begin and maintain_end respectively
- **description**: The instance description, assigned by referencing the input variable description
- **seconds_level_monitoring_enabled**, **seconds_level_monitoring_period**: The seconds-level monitoring switch and collection period, assigned by referencing the corresponding input variables respectively
- **audit_log_enabled**, **audit_log_keep_days**, **reserve_audit_logs**: The audit log configurations, assigned by referencing the corresponding input variables respectively
- **datastore**: The database engine information; the engine is fixed to `gaussdb-mysql` and the version is fixed to `8.0`
- **backup_strategy**: The backup strategy; `start_time` references the input variable instance_backup_time_window and `keep_days` references the input variable instance_backup_keep_days
- **tags**: The instance tags, assigned by referencing the input variable tags

### 7. Create an LTS Log Group and Log Stream

Add the following script to the TF file (such as main.tf):

```hcl
# Create an LTS log group and log stream in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "lts_group_name" {
  description = "The name of the LTS log group"
  type        = string
}

variable "lts_group_ttl_in_days" {
  description = "The expiration time (in days) of the LTS log group. The value is range from 1 to 365"
  type        = number
  default     = 30
}

variable "lts_stream_name" {
  description = "The name of the LTS log stream"
  type        = string
}

variable "lts_stream_ttl_in_days" {
  description = "The expiration time (in days) of the LTS log stream. The valid value is from 1 to 365"
  type        = number
  default     = 7
}

variable "lts_stream_is_favorite" {
  description = "Whether to favorite the log stream"
  type        = bool
  default     = false
}

resource "huaweicloud_lts_group" "test" {
  group_name  = var.lts_group_name
  ttl_in_days = var.lts_group_ttl_in_days
}

resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.lts_stream_name
  ttl_in_days = var.lts_stream_ttl_in_days
  is_favorite = var.lts_stream_is_favorite
}
```

**Parameter Description**:
- **group_name**: The log group name, assigned by referencing the input variable lts_group_name
- **ttl_in_days**: The log retention days of the log group, assigned by referencing the input variable lts_group_ttl_in_days
- **group_id**: The ID of the log group to which the log stream belongs, assigned by referencing the ID of the log group created above
- **stream_name**: The log stream name, assigned by referencing the input variable lts_stream_name
- **ttl_in_days**: The log retention days of the log stream, assigned by referencing the input variable lts_stream_ttl_in_days
- **is_favorite**: Whether to favorite the log stream, assigned by referencing the input variable lts_stream_is_favorite

### 8. Associate the TaurusDB Instance with the LTS Log Stream

Add the following script to the TF file (such as main.tf):

```hcl
# Associate the TaurusDB instance with the LTS log stream in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "lts_log_type" {
  description = "The type of the TaurusDB LTS log. Valid values are error_log, slow_log"
  type        = string
  default     = "error_log"
}

resource "huaweicloud_taurusdb_lts_log" "test" {
  instance_id   = huaweicloud_taurusdb_instance.test.id
  log_type      = var.lts_log_type
  lts_group_id  = huaweicloud_lts_group.test.id
  lts_stream_id = huaweicloud_lts_stream.test.id
}
```

**Parameter Description**:
- **instance_id**: The TaurusDB instance ID, assigned by referencing the ID of the TaurusDB instance created above
- **log_type**: The log type, assigned by referencing the input variable lts_log_type; valid values are `error_log` and `slow_log`
- **lts_group_id**: The LTS log group ID, assigned by referencing the ID of the log group created above
- **lts_stream_id**: The LTS log stream ID, assigned by referencing the ID of the log stream created above

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication parameters
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource parameters
vpc_name                    = "your_vpc"
subnet_name                 = "your_subnet"
security_group_name         = "your_security_group"
instance_name               = "your_taurusdb_instance"
instance_backup_time_window = "02:00-03:00"
instance_backup_keep_days   = 7
lts_group_name              = "your_lts_group"
lts_stream_name             = "your_lts_stream"
tags = {
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
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the TaurusDB instance and its associated LTS log
4. Run `terraform show` to view the created TaurusDB instance and its associated LTS log

## Reference Information

- [Huawei Cloud TaurusDB Product Documentation](https://support.huaweicloud.com/taurusdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For TaurusDB Instance with LTS Log](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/taurusdb/taurusdb-instance-with-lts-log)
