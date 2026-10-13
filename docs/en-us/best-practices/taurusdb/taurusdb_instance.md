# Deploy TaurusDB Instance

## Application Scenario

TaurusDB is an enterprise-grade cloud-native database service provided by Huawei Cloud. It is fully compatible with the MySQL protocol and adopts a compute-storage separation architecture, supporting one writer with multiple readers, second-level elastic scaling, and parallel query capabilities. It is suitable for online transaction processing (OLTP) scenarios with high requirements on performance, reliability, and scalability.

This best practice will introduce how to use Terraform to automatically deploy a TaurusDB instance, including the creation of VPC, subnet, security group, parameter template, instance, account, and database, helping you quickly build a usable cloud-native database environment.

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
- [TaurusDB Parameter Template (huaweicloud_taurusdb_parameter_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_parameter_template)
- [TaurusDB Instance (huaweicloud_taurusdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_instance)
- [TaurusDB Account (huaweicloud_taurusdb_account)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_account)
- [TaurusDB Database (huaweicloud_taurusdb_database)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_database)

### Resource/Data Source Dependencies

```
data.huaweicloud_taurusdb_flavors
    └── huaweicloud_taurusdb_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_taurusdb_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_taurusdb_instance

random_password
    ├── huaweicloud_taurusdb_instance
    └── huaweicloud_taurusdb_account

huaweicloud_taurusdb_parameter_template
    └── huaweicloud_taurusdb_instance

huaweicloud_taurusdb_instance
    ├── huaweicloud_taurusdb_account
    └── huaweicloud_taurusdb_database
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a VPC

Add the following script in the TF file (such as main.tf) to create a VPC:

```hcl
# Create a VPC resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The VPC name"
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

### 3. Create a Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create a subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet; when the input variable subnet_cidr is empty, the subnet CIDR is automatically calculated based on the VPC CIDR
- **gateway_ip**: The gateway IP address of the subnet; when the input variable gateway_ip is empty, the gateway IP is automatically calculated based on the subnet CIDR

### 4. Query TaurusDB Flavors

Add the following script in the TF file (such as main.tf) to query TaurusDB flavor information:

```hcl
# Query the TaurusDB flavors data source in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  # Get the first available AZ from the flavor's az_status
  available_azs = try([for k, v in data.huaweicloud_taurusdb_flavors.test.flavors[0].az_status : k if v == "normal"], [])
  master_az     = var.master_availability_zone != "" ? var.master_availability_zone : try(local.available_azs[0], "")
}
```

**Parameter Description**:
- **engine**: The database engine, fixed to gaussdb-mysql
- **version**: The database version, fixed to 8.0
- **availability_zone_mode**: The availability zone mode, assigned by referencing the input variable availability_zone_mode
- **locals.master_az**: The master node availability zone; when the input variable master_availability_zone is empty, the first available AZ is automatically selected from the flavor's AZ status

### 5. Create a Security Group

Add the following script in the TF file (such as main.tf) to create a security group and its rule:

```hcl
# Create a security group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **delete_default_rules**: Whether to delete the default rules, set to true to delete the default rules
- **security_group_id**: The ID of the security group to which the rule belongs, assigned by referencing the ID of the security group created in the previous step
- **direction**: The rule direction, set to ingress
- **ethertype**: The IP protocol type, set to IPv4
- **remote_ip_prefix**: The remote IP address range, assigned by referencing the input variable vpc_cidr
- **ports**: The port range, assigned by referencing the input variable instance_db_port
- **protocol**: The protocol type, set to tcp

### 6. Create a Random Password

Add the following script in the TF file (such as main.tf) to create a random password:

```hcl
# Create a random password resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **count**: Creates a random password when the input variable instance_password is empty, otherwise it is not created
- **length**: The password length, set to 12
- **special**: Whether to include special characters, set to true
- **override_special**: The set of allowed special characters
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters respectively

### 7. Create a TaurusDB Parameter Template

Add the following script in the TF file (such as main.tf) to create a TaurusDB parameter template:

```hcl
# Create a TaurusDB parameter template resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "configuration_id" {
  description = "The ID of an existing parameter template. If not specified, a new parameter template will be created"
  type        = string
  default     = ""
}

variable "parameter_template_name" {
  description = "The name of the parameter template to create"
  type        = string
  default     = ""
}

resource "huaweicloud_taurusdb_parameter_template" "test" {
  count = var.configuration_id == "" ? 1 : 0

  name              = var.parameter_template_name
  datastore_engine  = "gaussdb-mysql"
  datastore_version = "8.0"

  parameter_values = {
    auto_increment_increment = "100"
    character_set_server     = "gbk"
  }
}
```

**Parameter Description**:
- **count**: Creates a parameter template when the input variable configuration_id is empty, otherwise it is not created
- **name**: The parameter template name, assigned by referencing the input variable parameter_template_name
- **datastore_engine**: The database engine, fixed to gaussdb-mysql
- **datastore_version**: The database version, fixed to 8.0
- **parameter_values**: The parameter values in the parameter template; the example sets the auto-increment step and the server character set

### 8. Create a TaurusDB Instance

Add the following script in the TF file (such as main.tf) to create a TaurusDB instance:

```hcl
# Create a TaurusDB instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  flavor                           = var.instance_flavor_ref != "" ? var.instance_flavor_ref : try(data.huaweicloud_taurusdb_flavors.test.flavors[0].name, "")
  vpc_id                           = huaweicloud_vpc.test.id
  subnet_id                        = huaweicloud_vpc_subnet.test.id
  security_group_id                = huaweicloud_networking_secgroup.test.id
  password                         = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
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
  configuration_id                 = var.configuration_id != "" ? var.configuration_id : try(huaweicloud_taurusdb_parameter_template.test[0].id, null)
  maintain_begin                   = var.maintain_begin
  maintain_end                     = var.maintain_end
  description                      = var.description
  seconds_level_monitoring_enabled = var.seconds_level_monitoring_enabled
  seconds_level_monitoring_period  = var.seconds_level_monitoring_enabled ? var.seconds_level_monitoring_period : null

  datastore {
    engine  = "gaussdb-mysql"
    version = "8.0"
  }

  audit_log_enabled   = var.audit_log_enabled
  audit_log_keep_days = var.audit_log_keep_days
  reserve_audit_logs  = var.reserve_audit_logs

  backup_strategy {
    start_time = var.instance_backup_time_window
    keep_days  = tostring(var.instance_backup_keep_days)
  }

  tags = var.tags

  lifecycle {
    ignore_changes = [
      password, reserve_audit_logs, ssl_option, datastore[0].version,
    ]
  }
}
```

**Parameter Description**:
- **name**: The TaurusDB instance name, assigned by referencing the input variable instance_name
- **flavor**: The instance flavor; when the input variable instance_flavor_ref is empty, the first available flavor queried from the data source is used
- **vpc_id**: The ID of the VPC to which the instance belongs, assigned by referencing the ID of the VPC created in the previous step
- **subnet_id**: The ID of the subnet to which the instance belongs, assigned by referencing the ID of the subnet created in the previous step
- **security_group_id**: The ID of the security group to which the instance belongs, assigned by referencing the ID of the security group created in the previous step
- **password**: The instance password; when the input variable instance_password is empty, the random password is used
- **mode**: The instance mode, assigned by referencing the input variable instance_mode
- **availability_zone_mode**: The availability zone mode, assigned by referencing the input variable availability_zone_mode
- **master_availability_zone**: The master node availability zone, assigned by referencing the local variable master_az
- **read_replicas**: The number of read replicas, assigned by referencing the input variable read_replicas
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **volume_type**: The storage type, assigned by referencing the input variable volume_type
- **time_zone**: The time zone, assigned by referencing the input variable time_zone
- **port**: The database port, assigned by referencing the input variable instance_db_port
- **ssl_option**: Whether to enable SSL, assigned by referencing the input variable ssl_option
- **sql_filter_enabled**: Whether to enable SQL filter, assigned by referencing the input variable sql_filter_enabled
- **slow_log_show_original_switch**: Whether to enable slow log show original switch, assigned by referencing the input variable slow_log_show_original_switch
- **table_name_case_sensitivity**: Whether the kernel table name is case sensitive, assigned by referencing the input variable table_name_case_sensitivity
- **multi_tenant_switch**: Whether to enable multi-tenancy switch, assigned by referencing the input variable multi_tenant_switch
- **configuration_id**: The parameter template ID; when the input variable configuration_id is empty, the ID of the parameter template created in the previous step is used
- **maintain_begin**, **maintain_end**: The start and end time of the maintenance window, assigned by referencing the input variables maintain_begin and maintain_end
- **description**: The instance description, assigned by referencing the input variable description
- **seconds_level_monitoring_enabled**: Whether to enable seconds level monitoring, assigned by referencing the input variable seconds_level_monitoring_enabled
- **seconds_level_monitoring_period**: The seconds level monitoring collection period, assigned by referencing the input variable seconds_level_monitoring_period
- **datastore**: The database engine information, fixed to gaussdb-mysql 8.0
- **audit_log_enabled**: Whether to enable audit log, assigned by referencing the input variable audit_log_enabled
- **audit_log_keep_days**: The number of days for storing audit logs, assigned by referencing the input variable audit_log_keep_days
- **reserve_audit_logs**: Whether to reserve historical audit logs, assigned by referencing the input variable reserve_audit_logs
- **backup_strategy**: The backup strategy, where start_time is assigned by referencing the input variable instance_backup_time_window and keep_days is assigned by referencing the input variable instance_backup_keep_days
- **tags**: The instance tags, assigned by referencing the input variable tags

### 9. Create a TaurusDB Account

Add the following script in the TF file (such as main.tf) to create a TaurusDB account:

```hcl
# Create a TaurusDB account resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "account_name" {
  description = "Username with elevated privileges"
  type        = string
}

resource "huaweicloud_taurusdb_account" "test" {
  instance_id = huaweicloud_taurusdb_instance.test.id
  name        = var.account_name
  password    = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
}
```

**Parameter Description**:
- **instance_id**: The ID of the instance to which the account belongs, assigned by referencing the ID of the instance created in the previous step
- **name**: The account name, assigned by referencing the input variable account_name
- **password**: The account password; when the input variable instance_password is empty, the random password is used

### 10. Create a TaurusDB Database

Add the following script in the TF file (such as main.tf) to create a TaurusDB database:

```hcl
# Create a TaurusDB database resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "database_name" {
  description = "The name of the initial database"
  type        = string
}

variable "character_set" {
  description = "The character set of the database"
  type        = string
  default     = "utf8"
}

resource "huaweicloud_taurusdb_database" "test" {
  instance_id   = huaweicloud_taurusdb_instance.test.id
  name          = var.database_name
  character_set = var.character_set
}
```

**Parameter Description**:
- **instance_id**: The ID of the instance to which the database belongs, assigned by referencing the ID of the instance created in the previous step
- **name**: The database name, assigned by referencing the input variable database_name
- **character_set**: The database character set, assigned by referencing the input variable character_set

### 11. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name                    = "your_vpc"
subnet_name                 = "your_subnet"
security_group_name         = "your_security_group"
instance_name               = "your_taurusdb_instance"
account_name                = "your_account"
database_name               = "your_database"
instance_backup_time_window = "02:00-03:00"
instance_backup_keep_days   = 7
parameter_template_name     = "your_parameter_template"
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
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the TaurusDB instance
4. Run `terraform show` to view the created TaurusDB instance

## Reference Information

- [Huawei Cloud TaurusDB Product Documentation](https://support.huaweicloud.com/taurusdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For TaurusDB Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/taurusdb/taurusdb-instance)
