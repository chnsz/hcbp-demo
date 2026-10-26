# Deploy TaurusDB Instance and Associate EIP

## Application Scenario

TaurusDB is an enterprise-grade cloud-native database service provided by Huawei Cloud. By default, a TaurusDB instance is deployed in a Virtual Private Cloud (VPC) and can only be accessed over the private network. When services need to connect to the database from a local data center, another network environment, or public network clients, you need to associate an Elastic IP (EIP) with the instance to obtain a public network access entry.

This best practice will introduce how to use Terraform to automatically deploy a TaurusDB instance and associate an EIP with it, including VPC and subnet creation, security group and rule configuration, flavor query, TaurusDB instance creation, and EIP creation or reuse and association. The script supports two modes: when no existing EIP address is specified, a new EIP is created and associated; when an existing EIP address is specified, that EIP is queried and associated with the instance.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [TaurusDB Flavors (data.huaweicloud_taurusdb_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_flavors)
- [Elastic IP (data.huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/vpc_eip)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [TaurusDB Instance (huaweicloud_taurusdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_instance)
- [TaurusDB Instance EIP Association (huaweicloud_taurusdb_eip_associate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_eip_associate)

### Resource/Data Source Dependencies

```
data.huaweicloud_taurusdb_flavors.test
    └── huaweicloud_taurusdb_instance.test

data.huaweicloud_vpc_eip.test
    └── huaweicloud_taurusdb_eip_associate.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_taurusdb_instance.test

huaweicloud_networking_secgroup.test
    ├── huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_taurusdb_instance.test

random_password.test
    └── huaweicloud_taurusdb_instance.test

huaweicloud_vpc_eip.test
    └── huaweicloud_taurusdb_eip_associate.test

huaweicloud_taurusdb_instance.test
    └── huaweicloud_taurusdb_eip_associate.test
```

## Operation Steps

### 1. Script Preparation

Prepare a TF file (such as main.tf) in the specified workspace for writing the current best practice script, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the article [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a VPC and Subnet

Add the following script to the TF file (such as main.tf):

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
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC created above
- **cidr** (subnet): The subnet CIDR block, automatically divided from the VPC CIDR block when not specified
- **gateway_ip**: The subnet gateway IP, automatically calculated from the subnet CIDR block when not specified

### 3. Create a Security Group and Security Group Rule

Add the following script to the TF file (such as main.tf):

```hcl
# Create security group and security group rule resources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **delete_default_rules**: Whether to delete the default rules of the security group, set to true
- **security_group_id**: The ID of the security group to which the rule belongs, referencing the ID of the security group created above
- **direction**: The rule direction, set to ingress
- **remote_ip_prefix**: The source CIDR block allowed to access, assigned by referencing the input variable vpc_cidr
- **ports**: The port allowed to access, assigned by referencing the input variable instance_db_port
- **protocol**: The protocol type, set to tcp

### 4. Query TaurusDB Flavors

Add the following script to the TF file (such as main.tf):

```hcl
# Query the TaurusDB flavors data source in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone_mode" {
  description = "The availability zone mode. Valid values are single, multi"
  type        = string
  default     = "multi"
}

data "huaweicloud_taurusdb_flavors" "test" {
  engine                 = "gaussdb-mysql"
  version                = "8.0"
  availability_zone_mode = var.availability_zone_mode
}
```

**Parameter Description**:
- **engine**: The database engine, set to gaussdb-mysql
- **version**: The database version, set to 8.0
- **availability_zone_mode**: The availability zone mode, assigned by referencing the input variable availability_zone_mode

### 5. Create or Query an Elastic IP

Add the following script to the TF file (such as main.tf):

```hcl
# Create or query an EIP resource/data source in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "associate_eip_address" {
  description = "The existing EIP address to associate with the TaurusDB instance. If not specified, a new EIP will be created"
  type        = string
  default     = ""
}

variable "eip_type" {
  description = "The EIP type"
  type        = string
  default     = "5_bgp"
}

variable "bandwidth_name" {
  description = "The bandwidth name"
  type        = string
  default     = ""
}

variable "bandwidth_size" {
  description = "The bandwidth size in Mbit/s"
  type        = number
  default     = 5
}

variable "bandwidth_share_type" {
  description = "The share type of the bandwidth. Valid values are PER, WHOLE"
  type        = string
  default     = "PER"
}

variable "bandwidth_charge_mode" {
  description = "The charge mode of the bandwidth. Valid values are traffic, bandwidth"
  type        = string
  default     = "traffic"
}

# Query the existing EIP when an existing EIP address is specified
data "huaweicloud_vpc_eip" "test" {
  count = var.associate_eip_address != "" ? 1 : 0

  public_ip = var.associate_eip_address
}

# Create a new EIP when no existing EIP address is specified
resource "huaweicloud_vpc_eip" "test" {
  count = var.associate_eip_address == "" ? 1 : 0

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.bandwidth_name
    size        = var.bandwidth_size
    share_type  = var.bandwidth_share_type
    charge_mode = var.bandwidth_charge_mode
  }
}
```

**Parameter Description**:
- **count**: The data source is created when an existing EIP address is specified, and the resource is created when no existing EIP address is specified
- **public_ip**: The public address of the existing EIP, assigned by referencing the input variable associate_eip_address
- **type**: The EIP type, assigned by referencing the input variable eip_type
- **name**: The bandwidth name, assigned by referencing the input variable bandwidth_name
- **size**: The bandwidth size, assigned by referencing the input variable bandwidth_size
- **share_type**: The bandwidth share type, assigned by referencing the input variable bandwidth_share_type
- **charge_mode**: The bandwidth charge mode, assigned by referencing the input variable bandwidth_charge_mode

### 6. Create a TaurusDB Instance

Add the following script to the TF file (such as main.tf):

```hcl
# Create a TaurusDB instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "master_availability_zone" {
  description = "The master availability zone of the TaurusDB instance. If not specified, the first available AZ from flavors will be used"
  type        = string
  default     = ""
}

variable "instance_password" {
  description = "The password for the TaurusDB instance"
  type        = string
  default     = ""
  sensitive   = true
}

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

locals {
  # Get the first available AZ from the flavor's az_status
  available_azs = try([for k, v in data.huaweicloud_taurusdb_flavors.test.flavors[0].az_status : k if v == "normal"], [])
  master_az     = var.master_availability_zone != "" ? var.master_availability_zone : try(local.available_azs[0], "")
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
- **name**: The instance name, assigned by referencing the input variable instance_name
- **flavor**: The instance flavor, using the first available flavor queried from the flavors data source when not specified
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC created above
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet created above
- **security_group_id**: The ID of the security group to which the instance belongs, referencing the ID of the security group created above
- **password**: The instance password, generated by the random password resource when not specified
- **mode**: The instance mode, assigned by referencing the input variable instance_mode
- **availability_zone_mode**: The availability zone mode, assigned by referencing the input variable availability_zone_mode
- **master_availability_zone**: The master availability zone, using the first available AZ queried from the flavors data source when not specified
- **read_replicas**: The number of read replicas, assigned by referencing the input variable read_replicas
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **volume_type**: The storage type, assigned by referencing the input variable volume_type
- **time_zone**: The time zone, assigned by referencing the input variable time_zone
- **port**: The database port, assigned by referencing the input variable instance_db_port
- **ssl_option**: Whether to enable SSL, assigned by referencing the input variable ssl_option
- **sql_filter_enabled**: Whether to enable SQL filter, assigned by referencing the input variable sql_filter_enabled
- **slow_log_show_original_switch**: Whether to enable the slow log show original switch, assigned by referencing the input variable slow_log_show_original_switch
- **table_name_case_sensitivity**: Whether the kernel table name is case sensitive, assigned by referencing the input variable table_name_case_sensitivity
- **multi_tenant_switch**: Whether to enable the multi-tenancy switch, assigned by referencing the input variable multi_tenant_switch
- **maintain_begin** / **maintain_end**: The start and end time of the maintenance window, assigned by referencing the input variables maintain_begin and maintain_end
- **description**: The instance description, assigned by referencing the input variable description
- **seconds_level_monitoring_enabled** / **seconds_level_monitoring_period**: The seconds-level monitoring switch and collection period, assigned by referencing the input variables
- **audit_log_enabled** / **audit_log_keep_days** / **reserve_audit_logs**: The audit log switch, retention days, and historical log retention policy, assigned by referencing the input variables
- **backup_strategy**: The backup strategy, start_time references the input variable instance_backup_time_window and keep_days references the input variable instance_backup_keep_days
- **tags**: The instance tags, assigned by referencing the input variable tags

### 7. Associate the EIP with the TaurusDB Instance

Add the following script to the TF file (such as main.tf):

```hcl
# Create a TaurusDB instance EIP association resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
locals {
  public_ip    = var.associate_eip_address != "" ? var.associate_eip_address : huaweicloud_vpc_eip.test[0].address
  public_ip_id = var.associate_eip_address != "" ? data.huaweicloud_vpc_eip.test[0].id : huaweicloud_vpc_eip.test[0].id
}

resource "huaweicloud_taurusdb_eip_associate" "test" {
  instance_id  = huaweicloud_taurusdb_instance.test.id
  public_ip    = local.public_ip
  public_ip_id = local.public_ip_id
}
```

**Parameter Description**:
- **instance_id**: The ID of the TaurusDB instance to which the EIP is associated, referencing the ID of the instance created above
- **public_ip**: The public IP address to associate, using the existing EIP address when specified, otherwise using the address of the newly created EIP
- **public_ip_id**: The public IP ID to associate, using the queried EIP ID when an existing EIP address is specified, otherwise using the ID of the newly created EIP

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

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
instance_backup_time_window = "02:00-03:00"
instance_backup_keep_days   = 7
bandwidth_name              = "your_bandwidth"

# Fill in when using an existing EIP; bandwidth_name is not required in this case
# associate_eip_address = "your_existing_eip_address"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the TaurusDB instance and associating the EIP
4. Run `terraform show` to view the created TaurusDB instance and the association

## Reference Information

- [Huawei Cloud TaurusDB Product Documentation](https://support.huaweicloud.com/taurusdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For TaurusDB Instance Associate EIP](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/taurusdb/taurusdb-instance-associate-eip)
