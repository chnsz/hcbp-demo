# Deploy TaurusDB Instance with Database Proxy

## Application Scenario

TaurusDB is an enterprise-grade cloud-native database service provided by Huawei Cloud, fully compatible with the MySQL protocol and built on a compute-storage separation architecture with one-writer-multiple-readers and second-level elastic scaling. The database proxy sits between applications and the TaurusDB instance, providing read/write splitting, connection pooling, transaction splitting, and access control, which significantly improves connection reuse and read throughput in high-concurrency scenarios.

This best practice will introduce how to use Terraform to automatically deploy a TaurusDB instance with a database proxy, including VPC and subnet creation, security group configuration, instance flavor and availability zone query, TaurusDB instance creation, proxy flavor query, and database proxy creation with node weights, access control, and proxy parameters.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [TaurusDB Instance Flavors (data.huaweicloud_taurusdb_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_flavors)
- [TaurusDB Proxy Flavors (data.huaweicloud_taurusdb_proxy_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_proxy_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [TaurusDB Instance (huaweicloud_taurusdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_instance)
- [TaurusDB Database Proxy (huaweicloud_taurusdb_proxy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_proxy)

### Resource/Data Source Dependencies

```
data.huaweicloud_taurusdb_flavors
    ├── huaweicloud_taurusdb_instance
    │       └── data.huaweicloud_taurusdb_proxy_flavors
    │               └── huaweicloud_taurusdb_proxy
    └── (local.available_azs)

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            ├── huaweicloud_taurusdb_instance
            └── huaweicloud_taurusdb_proxy

huaweicloud_networking_secgroup
    └── huaweicloud_taurusdb_instance

random_password
    └── huaweicloud_taurusdb_instance
```

## Operation Steps

### 1. Script Preparation

Prepare a TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

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
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr, defaulting to 192.168.0.0/16
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC created above
- **cidr** (subnet): The subnet CIDR block, assigned by referencing the input variable subnet_cidr; when empty, it is automatically calculated from the VPC CIDR
- **gateway_ip**: The subnet gateway IP, assigned by referencing the input variable gateway_ip; when empty, the first available IP of the subnet is used

### 3. Query TaurusDB Instance Flavors and Availability Zones

Add the following script to the TF file (such as main.tf):

```hcl
# Query TaurusDB instance flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "master_availability_zone" {
  description = "The master availability zone of the TaurusDB instance. If not specified, the first available AZ from flavors will be used"
  type        = string
  default     = ""
}

data "huaweicloud_taurusdb_flavors" "test" {
  engine                 = "gaussdb-mysql"
  version                = "8.0"
  availability_zone_mode = "multi"
}

locals {
  # Obtain availability zones from the flavor's az_status (TaurusDB AZs must come from flavors, not huaweicloud_availability_zones)
  available_azs = try([for k, v in data.huaweicloud_taurusdb_flavors.test.flavors[0].az_status : k if v == "normal"], [])
  master_az     = var.master_availability_zone != "" ? var.master_availability_zone : try(local.available_azs[0], "")
}
```

**Parameter Description**:
- **engine**: The database engine, fixed to gaussdb-mysql
- **version**: The database version, fixed to 8.0
- **availability_zone_mode**: The availability zone mode, set to multi to query multi-AZ flavors
- **master_az**: The master node availability zone, preferring the input variable master_availability_zone; when not specified, the first available AZ from the flavors is used

### 4. Create a Security Group

Add the following script to the TF file (such as main.tf):

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **delete_default_rules**: Whether to delete the default rules, set to true so that access rules can be customized as needed

### 5. Generate a Random Instance Password

Add the following script to the TF file (such as main.tf):

```hcl
# Generate a random instance password in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **count**: Creates a random password when the input variable instance_password is empty; otherwise the user-specified password is used
- **length**: The password length, set to 12
- **override_special**: The set of allowed special characters
- **min_upper/min_lower/min_numeric/min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters in the password

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

variable "read_replicas" {
  description = "The number of read replicas"
  type        = number
  default     = 4
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

variable "instance_db_port" {
  description = "The database port"
  type        = number
  default     = 3306
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
  mode                             = "Cluster"
  availability_zone_mode           = "multi"
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
- **password**: The instance password, preferring the input variable instance_password; when empty, the random password is used
- **flavor**: The instance flavor, preferring the input variable instance_flavor_ref; when not specified, the first flavor from the flavor query is used
- **vpc_id/subnet_id/security_group_id**: Reference the IDs of the VPC, subnet, and security group created above, respectively
- **mode**: The instance mode, set to Cluster to support the database proxy
- **availability_zone_mode**: The availability zone mode, set to multi to support multi-AZ deployment
- **master_availability_zone**: The master node availability zone, referencing local.master_az
- **read_replicas**: The number of read replicas, assigned by referencing the input variable read_replicas; at least 4 are recommended when using the database proxy
- **port**: The database port, assigned by referencing the input variable instance_db_port
- **datastore**: The database engine and version, fixed to gaussdb-mysql 8.0
- **backup_strategy**: The backup strategy; start_time and keep_days reference the input variables instance_backup_time_window and instance_backup_keep_days, respectively
- **tags**: The instance tags, assigned by referencing the input variable tags

### 7. Query Database Proxy Flavors

Add the following script to the TF file (such as main.tf):

```hcl
# Query database proxy flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_taurusdb_proxy_flavors" "test" {
  instance_id = huaweicloud_taurusdb_instance.test.id
}

locals {
  # Sort nodes by name for consistent weight assignment
  sort_nodes = tolist(values({ for node in huaweicloud_taurusdb_instance.test.nodes : node.name => node }))
}
```

**Parameter Description**:
- **instance_id**: The TaurusDB instance ID, referencing the ID of the instance created in the previous step
- **sort_nodes**: Sorts the instance nodes by name, used later to assign weights to the proxy master node and read-only nodes

### 8. Create a Database Proxy

Add the following script to the TF file (such as main.tf):

```hcl
# Create a database proxy in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "proxy_node_num" {
  description = "The number of proxy nodes"
  type        = number
  default     = 2
}

variable "proxy_name" {
  description = "The name of the database proxy"
  type        = string
  default     = ""
}

variable "proxy_mode" {
  description = "The type of the proxy. Valid values are readwrite, readonly. Defaults to readwrite"
  type        = string
  default     = "readwrite"
}

variable "route_mode" {
  description = "The routing policy of the proxy. Valid values are 0, 1, 2"
  type        = number
  default     = 1
}

variable "proxy_new_node_auto_add_status" {
  description = "Whether to automatically add new nodes to the proxy. Valid values are ON, OFF"
  type        = string
  default     = "OFF"
}

variable "proxy_new_node_weight" {
  description = "The weight of new nodes automatically added to the proxy"
  type        = number
  default     = 20
}

variable "proxy_port" {
  description = "The proxy port"
  type        = number
  default     = 3306
}

variable "proxy_transaction_split" {
  description = "Whether to enable transaction split. Valid values are ON, OFF. Defaults to OFF"
  type        = string
  default     = "OFF"
}

variable "proxy_consistence_mode" {
  description = "The consistency mode. Valid values are session, global, eventual. Defaults to eventual"
  type        = string
  default     = "eventual"
}

variable "proxy_connection_pool_type" {
  description = "The connection pool type. Valid values are SESSION, CLOSED. Defaults to CLOSED"
  type        = string
  default     = "CLOSED"
}

variable "proxy_open_access_control" {
  description = "Whether to enable access control"
  type        = bool
  default     = true
}

variable "access_control_type" {
  description = "The access control mode. Valid values are white, black"
  type        = string
  default     = "white"
}

variable "proxy_dns_name_prefix" {
  description = "The DNS name prefix for the proxy"
  type        = string
  default     = ""
}

variable "proxy_access_control_ip_list" {
  description = "The access control IP list for the proxy"
  type        = list(object({
    ip          = string
    description = string
  }))
  default     = []
}

variable "proxy_master_node_weight" {
  description = "The weight of the master node in the proxy"
  type        = number
  default     = 20
}

variable "proxy_readonly_node_weight" {
  description = "The weight of read-only nodes in the proxy"
  type        = number
  default     = 30
}

variable "proxy_parameters" {
  description = "The parameters for the proxy"
  type        = list(object({
    name      = string
    value     = string
    elem_type = string
  }))
  default     = []
}

resource "huaweicloud_taurusdb_proxy" "test" {
  instance_id              = huaweicloud_taurusdb_instance.test.id
  flavor                   = try(data.huaweicloud_taurusdb_proxy_flavors.test.flavor_groups[0].flavors[0].spec_code, "")
  node_num                 = var.proxy_node_num
  proxy_name               = var.proxy_name
  proxy_mode               = var.proxy_mode
  route_mode               = var.route_mode
  subnet_id                = huaweicloud_vpc_subnet.test.id
  new_node_auto_add_status = var.proxy_new_node_auto_add_status
  new_node_weight          = var.proxy_new_node_weight
  port                     = var.proxy_port
  transaction_split        = var.proxy_transaction_split
  consistence_mode         = var.proxy_consistence_mode
  connection_pool_type     = var.proxy_connection_pool_type
  open_access_control      = var.proxy_open_access_control
  access_control_type      = var.proxy_open_access_control ? var.access_control_type : null
  dns_name_prefix          = var.proxy_dns_name_prefix

  dynamic "access_control_ip_list" {
    for_each = var.proxy_access_control_ip_list

    content {
      ip          = access_control_ip_list.value.ip
      description = access_control_ip_list.value.description
    }
  }

  master_node_weight {
    id     = try(local.sort_nodes[0].id, "")
    weight = var.proxy_master_node_weight
  }

  dynamic "readonly_nodes_weight" {
    for_each = length(local.sort_nodes) > 1 ? slice(local.sort_nodes, 1, length(local.sort_nodes)) : []

    content {
      id     = readonly_nodes_weight.value.id
      weight = var.proxy_readonly_node_weight
    }
  }

  dynamic "parameters" {
    for_each = var.proxy_parameters

    content {
      name      = parameters.value.name
      value     = parameters.value.value
      elem_type = parameters.value.elem_type
    }
  }

  lifecycle {
    ignore_changes = [
      new_node_weight, proxy_mode, readonly_nodes_weight, parameters,
    ]
  }
}
```

**Parameter Description**:
- **instance_id**: The ID of the TaurusDB instance to which the proxy belongs, referencing the ID of the instance created in the previous step
- **flavor**: The proxy flavor, referencing the first flavor from the proxy flavor query
- **node_num**: The number of proxy nodes, assigned by referencing the input variable proxy_node_num
- **proxy_name**: The proxy name, assigned by referencing the input variable proxy_name
- **proxy_mode**: The proxy mode, assigned by referencing the input variable proxy_mode; readwrite and readonly are supported
- **route_mode**: The routing policy, assigned by referencing the input variable route_mode
- **subnet_id**: The ID of the subnet to which the proxy belongs, referencing the ID of the subnet created above
- **new_node_auto_add_status/new_node_weight**: The new node auto-add status and new node weight, referencing the input variables proxy_new_node_auto_add_status and proxy_new_node_weight, respectively
- **port**: The proxy port, assigned by referencing the input variable proxy_port
- **transaction_split**: Whether to enable transaction splitting, assigned by referencing the input variable proxy_transaction_split
- **consistence_mode**: The consistency mode, assigned by referencing the input variable proxy_consistence_mode
- **connection_pool_type**: The connection pool type, assigned by referencing the input variable proxy_connection_pool_type
- **open_access_control/access_control_type**: Whether to enable access control and the access control mode, referencing the input variables proxy_open_access_control and access_control_type, respectively
- **dns_name_prefix**: The proxy DNS name prefix, assigned by referencing the input variable proxy_dns_name_prefix
- **access_control_ip_list**: The access control IP list, assigned by referencing the input variable proxy_access_control_ip_list
- **master_node_weight**: The master node weight; id references the first sorted node and weight references the input variable proxy_master_node_weight
- **readonly_nodes_weight**: The read-only node weights; id references the remaining sorted nodes and weight references the input variable proxy_readonly_node_weight
- **parameters**: The proxy parameters, assigned by referencing the input variable proxy_parameters

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
vpc_name                     = "your_vpc"
subnet_name                  = "your_subnet"
security_group_name          = "your_security_group"
instance_name                = "your_taurusdb_instance"
instance_backup_time_window  = "09:00-10:00"
instance_backup_keep_days    = 7
proxy_name                   = "your_proxy"
proxy_node_num               = 2
proxy_port                   = 3339
proxy_access_control_ip_list = [
  {
    ip          = "3.3.3.3"
    description = "test description"
  }
]
proxy_parameters             = [
  {
    name      = "multiStatementType"
    value     = "Loose"
    elem_type = "system"
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the TaurusDB instance and database proxy
4. Run `terraform show` to view the created TaurusDB instance and database proxy

## Reference Information

- [Huawei Cloud TaurusDB Product Documentation](https://support.huaweicloud.com/taurusdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For TaurusDB Instance with Database Proxy](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/taurusdb/taurusdb-instance-with-proxy)
