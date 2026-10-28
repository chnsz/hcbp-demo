# Deploy TaurusDB Instance with HTAP StarRocks Instance

## Application Scenario

TaurusDB is an enterprise-grade cloud-native database service provided by Huawei Cloud, fully compatible with the MySQL protocol and built on a compute-storage separation architecture. It is suitable for online transaction processing (OLTP) and other scenarios with high requirements on performance, reliability, and scalability. When a business needs both high-performance transaction processing and real-time analytics on the same database instance, you can attach an HTAP StarRocks instance to a TaurusDB instance to achieve hybrid transactional and analytical processing (HTAP).

This best practice will introduce how to use Terraform to automatically deploy a TaurusDB instance with an HTAP StarRocks instance, including VPC and subnet creation, security group configuration, TaurusDB flavor and availability zone query, TaurusDB instance creation, HTAP StarRocks flavor and engine version query, and the creation and parameter configuration of the HTAP StarRocks instance.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [TaurusDB Flavors (data.huaweicloud_taurusdb_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_flavors)
- [TaurusDB HTAP Flavors (data.huaweicloud_taurusdb_htap_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_htap_flavors)
- [TaurusDB HTAP Datastores (data.huaweicloud_taurusdb_htap_datastores)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_htap_datastores)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [TaurusDB Instance (huaweicloud_taurusdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_instance)
- [TaurusDB HTAP StarRocks Instance (huaweicloud_taurusdb_htap_starrocks_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_htap_starrocks_instance)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_taurusdb_instance

data.huaweicloud_taurusdb_flavors
    └── huaweicloud_taurusdb_instance

data.huaweicloud_taurusdb_htap_flavors
    └── huaweicloud_taurusdb_htap_starrocks_instance

data.huaweicloud_taurusdb_htap_datastores
    └── huaweicloud_taurusdb_htap_starrocks_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_taurusdb_instance
            └── huaweicloud_taurusdb_htap_starrocks_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_taurusdb_instance
    └── huaweicloud_taurusdb_htap_starrocks_instance

random_password
    ├── huaweicloud_taurusdb_instance
    └── huaweicloud_taurusdb_htap_starrocks_instance
```

## Operation Steps

### 1. Script Preparation

Prepare a TF file (such as main.tf) in the specified workspace for writing the current best practice script. Ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a VPC and Subnet

Add the following script to the TF file (such as main.tf) to create the network required by the TaurusDB instance and the HTAP StarRocks instance:

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
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr, defaulting to `192.168.0.0/16` when not specified
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC created above
- **cidr** (subnet): The subnet CIDR block, automatically divided from the VPC CIDR block when the input variable subnet_cidr is not specified
- **gateway_ip**: The subnet gateway IP, automatically set to the first available IP of the subnet CIDR block when the input variable gateway_ip is not specified

### 3. Create a Security Group

Add the following script to the TF file (such as main.tf) to create the security group used by the TaurusDB instance and the HTAP StarRocks instance:

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name of the TaurusDB instance"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group, set to `true` so that access rules can be customized as needed

### 4. Generate a Random Password

Add the following script to the TF file (such as main.tf) to generate the initial passwords for the TaurusDB instance and the HTAP StarRocks instance:

```hcl
# Generate a random password for the TaurusDB instance and the HTAP StarRocks instance
resource "random_password" "test" {
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
- **length**: The password length, set to `16`
- **special**: Whether to include special characters, set to `true`
- **override_special**: The set of allowed special characters
- **min_upper** / **min_lower** / **min_numeric** / **min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters respectively

### 5. Query Availability Zones, TaurusDB Flavors, and HTAP Engine Resources

Add the following script to the TF file (such as main.tf) to query the availability zones, instance flavors, and engine versions required for deployment:

```hcl
# Query availability zones, TaurusDB flavors, and HTAP engine resources
variable "taurusdb_availability_zone_mode" {
  description = "The availability zone mode of the TaurusDB instance. Valid values are single, multi"
  type        = string
  default     = "multi"
}

variable "az_code" {
  description = "The AZ code of the HTAP StarRocks instance. If not specified, the first available AZ from flavors will be used"
  type        = string
  default     = ""
}

variable "fe_flavor_id" {
  description = "The specification ID of the frontend node. If not specified, the first available flavor will be used"
  type        = string
  default     = ""
}

variable "be_flavor_id" {
  description = "The specification ID of the backend node. If not specified, the first available flavor will be used"
  type        = string
  default     = ""
}

variable "engine_version" {
  description = "The major version number of the engine. If not specified, the latest version from the data source will be used"
  type        = string
  default     = ""
}

data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_taurusdb_flavors" "test" {
  engine                 = "gaussdb-mysql"
  version                = "8.0"
  availability_zone_mode = var.taurusdb_availability_zone_mode
}

data "huaweicloud_taurusdb_htap_flavors" "test" {
  engine_name            = "star-rocks"
  availability_zone_mode = "single"
}

data "huaweicloud_taurusdb_htap_datastores" "test" {
  engine_name = "star-rocks"
}

locals {
  # Get the first available AZ from the az_status of the HTAP flavor
  htap_az_status = try(data.huaweicloud_taurusdb_htap_flavors.test.flavors[0].az_status, {})
  all_az_normal  = [for k, v in local.htap_az_status : k if v == "normal"]
  az_code        = var.az_code != "" ? var.az_code : try(local.all_az_normal[0], "")

  # Filter the frontend and backend flavors by spec code and AZ status
  be_flavors = [for f in data.huaweicloud_taurusdb_htap_flavors.test.flavors : f if length(regexall("sr-be", f.spec_code)) > 0 && contains(keys(f.az_status), local.az_code) && f.az_status[local.az_code] == "normal"]
  fe_flavors = [for f in data.huaweicloud_taurusdb_htap_flavors.test.flavors : f if length(regexall("sr-fe", f.spec_code)) > 0 && contains(keys(f.az_status), local.az_code) && f.az_status[local.az_code] == "normal"]

  fe_flavor_id   = var.fe_flavor_id != "" ? var.fe_flavor_id : try(local.fe_flavors[0].id, "")
  be_flavor_id   = var.be_flavor_id != "" ? var.be_flavor_id : try(local.be_flavors[0].id, "")
  engine_version = var.engine_version != "" ? var.engine_version : try(data.huaweicloud_taurusdb_htap_datastores.test.datastores[0].name, "")
}
```

**Parameter Description**:
- **engine**: The TaurusDB instance engine, set to `gaussdb-mysql`
- **version**: The TaurusDB instance engine version, set to `8.0`
- **availability_zone_mode**: The TaurusDB instance availability zone mode, assigned by referencing the input variable taurusdb_availability_zone_mode
- **engine_name**: The HTAP instance engine name, set to `star-rocks`
- **locals.az_code**: The AZ code of the HTAP StarRocks instance; when the input variable az_code is not specified, the first AZ with status `normal` is selected automatically
- **locals.fe_flavor_id** / **locals.be_flavor_id**: The frontend/backend node flavor IDs of the HTAP StarRocks instance; when not specified through input variables, available flavors are filtered automatically from the data source results
- **locals.engine_version**: The HTAP StarRocks instance engine version; when the input variable engine_version is not specified, the latest version returned by the data source is used

### 6. Create a TaurusDB Instance

Add the following script to the TF file (such as main.tf) to create a TaurusDB instance:

```hcl
# Create a TaurusDB instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "taurusdb_instance_name" {
  description = "The TaurusDB instance name"
  type        = string
}

variable "taurusdb_flavor_ref" {
  description = "The flavor code of the TaurusDB instance. If not specified, the first available flavor will be used"
  type        = string
  default     = ""
}

variable "taurusdb_root_pwd" {
  description = "The database password of the TaurusDB instance. If not specified, a random password will be generated"
  type        = string
  default     = ""
  sensitive   = true
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = "0"
}

variable "taurusdb_read_replicas" {
  description = "The number of read replicas of the TaurusDB instance"
  type        = number
  default     = 2
}

resource "huaweicloud_taurusdb_instance" "test" {
  name                     = var.taurusdb_instance_name
  flavor                   = var.taurusdb_flavor_ref != "" ? var.taurusdb_flavor_ref : try(data.huaweicloud_taurusdb_flavors.test.flavors[0].name, "")
  vpc_id                   = huaweicloud_vpc.test.id
  subnet_id                = huaweicloud_vpc_subnet.test.id
  security_group_id        = huaweicloud_networking_secgroup.test.id
  password                 = var.taurusdb_root_pwd != "" ? var.taurusdb_root_pwd : random_password.test.result
  enterprise_project_id    = var.enterprise_project_id
  availability_zone_mode   = var.taurusdb_availability_zone_mode
  master_availability_zone = try(data.huaweicloud_availability_zones.test.names[0], "")
  read_replicas            = var.taurusdb_read_replicas

  datastore {
    engine  = "gaussdb-mysql"
    version = "8.0"
  }

  lifecycle {
    ignore_changes = [
      password, datastore[0].version,
    ]
  }
}
```

**Parameter Description**:
- **name**: The TaurusDB instance name, assigned by referencing the input variable taurusdb_instance_name
- **flavor**: The TaurusDB instance flavor; when the input variable taurusdb_flavor_ref is not specified, the first available flavor returned by the data source is used
- **vpc_id** / **subnet_id**: The VPC and subnet to which the instance belongs, assigned by referencing the IDs of the VPC and subnet created above
- **security_group_id**: The security group bound to the instance, assigned by referencing the ID of the security group created above
- **password**: The initial password of the instance; when the input variable taurusdb_root_pwd is not specified, the randomly generated password is used
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **availability_zone_mode**: The instance availability zone mode, assigned by referencing the input variable taurusdb_availability_zone_mode
- **master_availability_zone**: The AZ of the primary node, assigned by referencing the first AZ name returned by the availability zones data source
- **read_replicas**: The number of read replicas, assigned by referencing the input variable taurusdb_read_replicas
- **datastore.engine** / **datastore.version**: The instance engine and version, set to `gaussdb-mysql` and `8.0` respectively

### 7. Create an HTAP StarRocks Instance

Add the following script to the TF file (such as main.tf) to create an HTAP StarRocks instance attached to the TaurusDB instance:

```hcl
# Create an HTAP StarRocks instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "htap_instance_name" {
  description = "The HTAP StarRocks instance name. The name must start with a letter and consist of 4 to 64 characters"
  type        = string
}

variable "fe_count" {
  description = "The number of frontend nodes. For a single-node instance, the value is fixed to 1. For a cluster instance, the value ranges from 3 to 10"
  type        = number
  default     = 1
}

variable "be_count" {
  description = "The number of backend nodes. For a single-node instance, the value is fixed to 1. For a cluster instance, the value ranges from 3 to 10"
  type        = number
  default     = 1
}

variable "htap_db_root_pwd" {
  description = "The database password of the HTAP StarRocks instance. If not specified, a random password will be generated"
  type        = string
  default     = ""
  sensitive   = true
}

variable "time_zone" {
  description = "The time zone of the HTAP StarRocks instance"
  type        = string
  default     = "UTC+08:00"
}

variable "enable_users_sync" {
  description = "Whether to enable users synchronization. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "open_slow_log_switch" {
  description = "Whether to enable the slow query log original text switch. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "ha_mode" {
  description = "The deployment mode of the HTAP StarRocks instance. Valid values are Single, Cluster"
  type        = string
  default     = "Single"
}

variable "volume_io_type" {
  description = "The storage type of the frontend and backend nodes. Valid values are SSD, ESSD"
  type        = string
  default     = "SSD"
}

variable "fe_volume_capacity" {
  description = "The disk capacity in GB of the frontend node. The value ranges from 50 to 1000 and the increment is 10 GB"
  type        = number
  default     = 50
}

variable "be_volume_capacity" {
  description = "The disk capacity in GB of the backend node. The value ranges from 50 to 1000 and the increment is 10 GB"
  type        = number
  default     = 50
}

variable "be_parameter_values" {
  description = "A map contains mappings of parameter name and value to modify for the backend nodes"
  type        = map(string)
  default     = {}
}

variable "fe_parameter_values" {
  description = "A map contains mappings of parameter name and value to modify for the frontend nodes"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_taurusdb_htap_starrocks_instance" "test" {
  instance_id          = huaweicloud_taurusdb_instance.test.id
  name                 = var.htap_instance_name
  fe_flavor_id         = local.fe_flavor_id
  be_flavor_id         = local.be_flavor_id
  az_code              = local.az_code
  az_mode              = "single"
  fe_count             = var.fe_count
  be_count             = var.be_count
  db_root_pwd          = var.htap_db_root_pwd != "" ? var.htap_db_root_pwd : random_password.test.result
  time_zone            = var.time_zone
  security_group_id    = huaweicloud_networking_secgroup.test.id
  enable_users_sync    = var.enable_users_sync
  open_slow_log_switch = var.open_slow_log_switch

  engine {
    type    = "star-rocks"
    version = local.engine_version
  }

  ha {
    mode = var.ha_mode
  }

  fe_volume {
    io_type        = var.volume_io_type
    capacity_in_gb = var.fe_volume_capacity
  }

  be_volume {
    io_type        = var.volume_io_type
    capacity_in_gb = var.be_volume_capacity
  }

  tags_info {
    sys_tags {
      key   = "_sys_enterprise_project_id"
      value = var.enterprise_project_id
    }
  }

  be_parameter_values = var.be_parameter_values
  fe_parameter_values = var.fe_parameter_values

  lifecycle {
    ignore_changes = [
      db_root_pwd, be_parameter_values, fe_parameter_values,
    ]
  }
}
```

**Parameter Description**:
- **instance_id**: The ID of the TaurusDB instance to which the HTAP StarRocks instance is attached, assigned by referencing the ID of the TaurusDB instance created above
- **name**: The HTAP StarRocks instance name, assigned by referencing the input variable htap_instance_name
- **fe_flavor_id** / **be_flavor_id**: The frontend/backend node flavor IDs, assigned by referencing the local variables local.fe_flavor_id and local.be_flavor_id
- **az_code**: The AZ code of the instance, assigned by referencing the local variable local.az_code
- **az_mode**: The availability zone mode, set to `single`
- **fe_count** / **be_count**: The number of frontend/backend nodes, assigned by referencing the input variables fe_count and be_count respectively
- **db_root_pwd**: The initial password of the instance; when the input variable htap_db_root_pwd is not specified, the randomly generated password is used
- **time_zone**: The time zone of the instance, assigned by referencing the input variable time_zone
- **security_group_id**: The security group bound to the instance, assigned by referencing the ID of the security group created above
- **enable_users_sync**: Whether to enable user synchronization, assigned by referencing the input variable enable_users_sync
- **open_slow_log_switch**: Whether to enable the slow query log original text switch, assigned by referencing the input variable open_slow_log_switch
- **engine.type** / **engine.version**: The instance engine type and version, set to `star-rocks` and the local variable local.engine_version respectively
- **ha.mode**: The instance deployment mode, assigned by referencing the input variable ha_mode
- **fe_volume** / **be_volume**: The storage configuration of the frontend/backend nodes, assigned by referencing the input variables volume_io_type, fe_volume_capacity, and be_volume_capacity respectively
- **tags_info.sys_tags**: The system tags of the instance, writing the enterprise project ID into the `_sys_enterprise_project_id` tag
- **be_parameter_values** / **fe_parameter_values**: The parameter configuration of the backend/frontend nodes, assigned by referencing the input variables be_parameter_values and fe_parameter_values respectively

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
vpc_name               = "your_vpc"
subnet_name            = "your_subnet"
security_group_name    = "your_security_group"
taurusdb_instance_name = "your_taurusdb_instance"
htap_instance_name     = "your_htap_starrocks_instance"
be_parameter_values    = {
  alter_tablet_worker_count            = "1"
  base_compaction_num_threads_per_disk = "1"
}
fe_parameter_values    = {
  alter_table_timeout_second     = "21600"
  bdbje_heartbeat_timeout_second = "10"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the TaurusDB instance and the HTAP StarRocks instance
4. Run `terraform show` to view the created TaurusDB instance and HTAP StarRocks instance

## Reference Information

- [Huawei Cloud TaurusDB Product Documentation](https://support.huaweicloud.com/taurusdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For TaurusDB Instance with HTAP StarRocks Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/taurusdb/taurusdb-instance-with-htap)
