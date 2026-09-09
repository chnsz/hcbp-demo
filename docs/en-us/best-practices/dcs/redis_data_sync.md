# Deploy Redis Data Synchronization

## Application Scenario

Distributed Cache Service (DCS) is a high-performance, highly available in-memory database service provided by Huawei Cloud, supporting mainstream cache engines such as Redis and Memcached. In scenarios such as business migration, disaster recovery, or data consolidation, it is often necessary to synchronize data from one Redis instance to another.

This best practice will introduce how to use Terraform to automatically deploy two DCS Redis instances and create full and incremental online data migration tasks to achieve data synchronization from the source instance to the target instance. Through this practice, you can quickly build a complete Redis data synchronization environment, including VPC, subnet, security group, DCS instances, and migration tasks.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS Flavors (data.huaweicloud_dcs_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DCS Instance (huaweicloud_dcs_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS Online Data Migration Task (huaweicloud_dcs_online_data_migration_task)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_online_data_migration_task)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_dcs_instance

huaweicloud_vpc_subnet
    └── huaweicloud_dcs_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dcs_instance

huaweicloud_dcs_instance
    └── huaweicloud_dcs_online_data_migration_task
```

## Operation Steps

### 1. Script Preparation

Prepare the TF files (such as main.tf) for writing the current best practice script in the specified working directory, and ensure that they (or other TF files in the same level directory) contain the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create VPC and Subnet

Add the following script to the TF file (such as main.tf):

```hcl
# Create VPC and subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
  default     = "dcs-sync-vpc"
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
  default     = "dcs-sync-subnet"
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
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
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable vpc_name, specifying the VPC name.
- **cidr**: Assigned by referencing the input variable vpc_cidr, specifying the CIDR block of the VPC.
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, specifying the VPC to which the subnet belongs.
- **name**: Assigned by referencing the input variable subnet_name, specifying the subnet name.
- **cidr**: Assigned by referencing the input variable subnet_cidr, if not set, it is automatically divided from the VPC CIDR.
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip, if not set, the gateway address is automatically calculated.

### 3. Create Security Group

Add the following script to the TF file (such as main.tf):

```hcl
# Create security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
  default     = "dcs-sync-sg"
}

resource "huaweicloud_networking_secgroup" "test" {
  name        = var.security_group_name
  description = "Security group for DCS data migration"
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable security_group_name, specifying the security group name.
- **description**: Description of the security group.

### 4. Query Availability Zones and DCS Flavors

Add the following script to the TF file (such as main.tf):

```hcl
# Query availability zones and DCS flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_dcs_flavors" "test" {
  cache_mode     = var.instance_cache_mode
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**Parameter Description**:
- **cache_mode**: Assigned by referencing the input variable instance_cache_mode, specifying the cache mode.
- **capacity**: Assigned by referencing the input variable instance_capacity, specifying the cache capacity (GB).
- **engine_version**: Assigned by referencing the input variable instance_engine_version, specifying the engine version.

### 5. Create Source and Target DCS Redis Instances

Add the following script to the TF file (such as main.tf):

```hcl
# Create source and target DCS Redis instances in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_cache_mode" {
  description = "The cache mode of the DCS instances"
  type        = string
  default     = "ha"
}

variable "instance_capacity" {
  description = "The capacity of the DCS instances (GB)"
  type        = number
  default     = 4
}

variable "instance_engine_version" {
  description = "The engine version of the DCS instances"
  type        = string
  default     = "5.0"
}

variable "instance_name" {
  description = "The base name of the DCS Redis instances (will be suffixed with -0 and -1)"
  type        = string
}

variable "instance_password" {
  description = "The password of the DCS instances"
  type        = string
  sensitive   = true
}

resource "huaweicloud_dcs_instance" "test" {
  count = 2

  name               = "${var.instance_name}-${count.index}"
  engine             = "Redis"
  engine_version     = var.instance_engine_version
  capacity           = var.instance_capacity
  flavor             = try(data.huaweicloud_dcs_flavors.test.flavors[0].name, null)
  availability_zones = try(slice(data.huaweicloud_availability_zones.test.names, 0, 2), null)
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  password           = var.instance_password

  lifecycle {
    ignore_changes = [
      security_group_id
    ]
  }
}
```

**Parameter Description**:
- **count**: Creates 2 instances, serving as the source and target instances respectively.
- **name**: Assigned by referencing the input variable instance_name and count.index, the instance names are `${instance_name}-0` and `${instance_name}-1`.
- **engine**: Cache engine, fixed to Redis.
- **engine_version**: Assigned by referencing the input variable instance_engine_version, specifying the engine version.
- **capacity**: Assigned by referencing the input variable instance_capacity, specifying the cache capacity (GB).
- **flavor**: Assigned by referencing data.huaweicloud_dcs_flavors.test.flavors[0].name, specifying the instance flavor.
- **availability_zones**: Assigned by referencing data.huaweicloud_availability_zones.test.names, specifying the availability zones.
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, specifying the VPC to which the instance belongs.
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id, specifying the subnet to which the instance belongs.
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id, specifying the security group.
- **password**: Assigned by referencing the input variable instance_password, specifying the instance password.

### 6. Create Full Migration Task

Add the following script to the TF file (such as main.tf):

```hcl
# Create full migration task in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "full_migration_task_name" {
  description = "The name of the full migration task"
  type        = string
  default     = "full-migration-task"
}

variable "full_migration_task_description" {
  description = "The description of the full migration task"
  type        = string
  default     = "Full data migration from source to target DCS instance"
}

variable "full_migration_resume_mode" {
  description = "The reconnection mode for full migration"
  type        = string
  default     = "auto"
}

variable "full_migration_bandwidth_limit_mb" {
  description = "The bandwidth limit for full migration (MB/s)"
  type        = string
  default     = ""
}

resource "huaweicloud_dcs_online_data_migration_task" "full_migration" {
  task_name          = var.full_migration_task_name
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  description        = var.full_migration_task_description
  migration_method   = "full_amount_migration"
  resume_mode        = var.full_migration_resume_mode
  bandwidth_limit_mb = var.full_migration_bandwidth_limit_mb != "" ? var.full_migration_bandwidth_limit_mb : null

  source_instance {
    id       = huaweicloud_dcs_instance.test[0].id
    password = var.instance_password
  }

  target_instance {
    id       = huaweicloud_dcs_instance.test[1].id
    password = var.instance_password
  }

  lifecycle {
    ignore_changes = [
      source_instance,
      target_instance
    ]
  }

  depends_on = [
    huaweicloud_dcs_instance.test
  ]
}
```

**Parameter Description**:
- **task_name**: Assigned by referencing the input variable full_migration_task_name, specifying the migration task name.
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, specifying the VPC to which the migration task belongs.
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id, specifying the subnet to which the migration task belongs.
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id, specifying the security group.
- **description**: Assigned by referencing the input variable full_migration_task_description, specifying the task description.
- **migration_method**: Migration method, fixed to `full_amount_migration`, indicating full migration.
- **resume_mode**: Assigned by referencing the input variable full_migration_resume_mode, specifying the resume mode.
- **bandwidth_limit_mb**: Assigned by referencing the input variable full_migration_bandwidth_limit_mb, specifying the bandwidth limit, no limit if empty.
- **source_instance**: Source instance configuration, assigned by referencing huaweicloud_dcs_instance.test[0].id and the input variable instance_password.
- **target_instance**: Target instance configuration, assigned by referencing huaweicloud_dcs_instance.test[1].id and the input variable instance_password.

### 7. Create Incremental Migration Task (Optional)

Add the following script to the TF file (such as main.tf):

```hcl
# Create incremental migration task in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "enable_incremental_migration" {
  description = "Whether to enable incremental migration after full migration"
  type        = bool
  default     = true
}

variable "incremental_migration_task_name" {
  description = "The name of the incremental migration task"
  type        = string
  default     = "incremental-migration-task"
}

variable "incremental_migration_task_description" {
  description = "The description of the incremental migration task"
  type        = string
  default     = "Incremental data migration from source to target DCS instance"
}

variable "incremental_migration_resume_mode" {
  description = "The reconnection mode for incremental migration"
  type        = string
  default     = "auto"
}

variable "incremental_migration_bandwidth_limit_mb" {
  description = "The bandwidth limit for incremental migration (MB/s)"
  type        = string
  default     = ""
}

resource "huaweicloud_dcs_online_data_migration_task" "incremental_migration" {
  count = var.enable_incremental_migration ? 1 : 0

  task_name          = var.incremental_migration_task_name
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  description        = var.incremental_migration_task_description
  migration_method   = "incremental_migration"
  resume_mode        = var.incremental_migration_resume_mode
  bandwidth_limit_mb = var.incremental_migration_bandwidth_limit_mb != "" ? var.incremental_migration_bandwidth_limit_mb : null

  source_instance {
    id       = huaweicloud_dcs_instance.test[0].id
    password = var.instance_password
  }

  target_instance {
    id       = huaweicloud_dcs_instance.test[1].id
    password = var.instance_password
  }

  lifecycle {
    ignore_changes = [
      source_instance,
      target_instance
    ]
  }

  depends_on = [
    huaweicloud_dcs_instance.test,
    huaweicloud_dcs_online_data_migration_task.full_migration
  ]
}
```

**Parameter Description**:
- **count**: Assigned by referencing the input variable enable_incremental_migration, when true, creates 1 incremental migration task, otherwise not created.
- **task_name**: Assigned by referencing the input variable incremental_migration_task_name, specifying the migration task name.
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, specifying the VPC to which the migration task belongs.
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id, specifying the subnet to which the migration task belongs.
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id, specifying the security group.
- **description**: Assigned by referencing the input variable incremental_migration_task_description, specifying the task description.
- **migration_method**: Migration method, fixed to `incremental_migration`, indicating incremental migration.
- **resume_mode**: Assigned by referencing the input variable incremental_migration_resume_mode, specifying the resume mode.
- **bandwidth_limit_mb**: Assigned by referencing the input variable incremental_migration_bandwidth_limit_mb, specifying the bandwidth limit, no limit if empty.
- **source_instance**: Source instance configuration, assigned by referencing huaweicloud_dcs_instance.test[0].id and the input variable instance_password.
- **target_instance**: Target instance configuration, assigned by referencing huaweicloud_dcs_instance.test[1].id and the input variable instance_password.

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a way to preset these configurations through `tfvars` files, avoiding repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# DCS Instance Configuration
instance_name     = "redis-instance"
instance_password = "YourPassword@123"

# Full Migration Task Configuration
full_migration_bandwidth_limit_mb = "100"

# Incremental Migration Task Configuration
enable_incremental_migration             = true
incremental_migration_bandwidth_limit_mb = "50"
```

**Usage**:

1. Save the above content as the `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in the file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="instance_name=my-instance"`
2. Environment variables: `export TF_VAR_instance_name=my-instance`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating DCS Redis instances and migration tasks
4. Run `terraform show` to view the created DCS Redis instances and migration tasks

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis Data Synchronization](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-data-sync)
