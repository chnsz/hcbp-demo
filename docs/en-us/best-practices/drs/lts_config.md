# Deploy LTS Configuration

## Application Scenario

Data Replication Service (DRS) is a one-stop data replication service provided by Huawei Cloud, supporting scenarios such as database cloud migration, database migration, real-time database synchronization, and database disaster recovery. During the running of migration or synchronization jobs, job logs are an important basis for troubleshooting data inconsistency and locating abnormal interruptions.

This best practice will introduce how to use Terraform to configure Log Tank Service (LTS) logging for a DRS migration job, delivering job logs to a specified log group and log stream in real time for subsequent retrieval, analysis, and long-term retention.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS Flavors (data.huaweicloud_rds_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RDS Instance (huaweicloud_rds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DRS Job (huaweicloud_drs_job)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_job)
- [LTS Log Group (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS Log Stream (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [DRS Job LTS Configuration (huaweicloud_drs_lts_config)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_lts_config)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_rds_instance

data.huaweicloud_rds_flavors
    └── huaweicloud_rds_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_rds_instance
            └── huaweicloud_drs_job
                └── huaweicloud_drs_lts_config

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_rds_instance

huaweicloud_lts_group
    └── huaweicloud_lts_stream
        └── huaweicloud_drs_lts_config
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [preparation before deploying Huawei Cloud resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create VPC and Subnet

Add the following script in the TF file (such as main.tf) to create a VPC and subnet:

```hcl
# Create VPC and subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  nullable    = false
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = false
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
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr, defaulting to `192.168.0.0/16`
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC resource
- **cidr**: The subnet CIDR block; when the input variable subnet_cidr is empty, the subnet is automatically derived from the VPC CIDR block
- **gateway_ip**: The subnet gateway IP; when the input variable gateway_ip is empty, it is automatically derived from the subnet CIDR block

### 3. Create Security Group and Rules

Add the following script in the TF file to create a security group and database access rules:

```hcl
# Create a security group and its rules in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = 2

  security_group_id = huaweicloud_networking_secgroup.test.id
  ethertype         = "IPv4"
  remote_ip_prefix  = "192.168.0.0/16"
  protocol          = "tcp"
  direction         = count.index == 0 ? "ingress" : "egress"
  ports             = count.index == 0 ? "3306" : null
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group, set to `true`
- **security_group_id**: The ID of the security group to which the rule belongs, assigned by referencing the ID of the security group resource
- **direction**: The rule direction; the first rule is ingress and the second is egress
- **ports**: The ingress rule opens port `3306` for MySQL database access

### 4. Query Availability Zones and RDS Flavors

Add the following script in the TF file to query availability zones and RDS flavor information:

```hcl
# Query availability zones and RDS flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "rds_db_type" {
  description = "The database type for querying RDS flavors"
  type        = string
  default     = "MySQL"
}

variable "rds_db_version" {
  description = "The database version for querying RDS flavors"
  type        = string
  default     = "5.7"
}

variable "rds_instance_mode" {
  description = "The instance mode for querying RDS flavors"
  type        = string
  default     = "ha"
}

data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_rds_flavors" "test" {
  db_type       = var.rds_db_type
  db_version    = var.rds_db_version
  instance_mode = var.rds_instance_mode
}
```

**Parameter Description**:
- **db_type**: The database type, assigned by referencing the input variable rds_db_type, defaulting to `MySQL`
- **db_version**: The database version, assigned by referencing the input variable rds_db_version, defaulting to `5.7`
- **instance_mode**: The instance mode, assigned by referencing the input variable rds_instance_mode, defaulting to `ha`

### 5. Create Source and Destination RDS Instances

Add the following script in the TF file to create source and destination RDS MySQL instances:

```hcl
# Create source and destination RDS instances in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "source_rds_name" {
  description = "The name of the source RDS instance"
  type        = string
}

variable "dest_rds_name" {
  description = "The name of the destination RDS instance"
  type        = string
}

variable "rds_flavor" {
  description = "The flavor of the RDS instances"
  type        = string
  default     = "rds.mysql.x1.large.2.ha"
}

variable "source_rds_fixed_ip" {
  description = "The fixed IP address of the source RDS instance"
  type        = string
}

variable "dest_rds_fixed_ip" {
  description = "The fixed IP address of the destination RDS instance"
  type        = string
}

variable "db_password" {
  description = "The password for the RDS root user and DRS database connections"
  type        = string
  sensitive   = true
}

resource "huaweicloud_rds_instance" "test" {
  count = 2

  name                = count.index == 0 ? var.source_rds_name : var.dest_rds_name
  flavor              = var.rds_flavor != "" ? var.rds_flavor : try(data.huaweicloud_rds_flavors.test.flavors[0].name, null)
  security_group_id   = huaweicloud_networking_secgroup.test.id
  subnet_id           = huaweicloud_vpc_subnet.test.id
  vpc_id              = huaweicloud_vpc.test.id
  fixed_ip            = count.index == 0 ? var.source_rds_fixed_ip : var.dest_rds_fixed_ip
  ha_replication_mode = "semisync"

  availability_zone = [
    try(data.huaweicloud_availability_zones.test.names[0], ""),
    try(data.huaweicloud_availability_zones.test.names[3], ""),
  ]

  db {
    password = var.db_password
    type     = "MySQL"
    version  = "5.7"
    port     = 3306
  }

  volume {
    type = "CLOUDSSD"
    size = 40
  }
}
```

**Parameter Description**:
- **count**: Creates two RDS instances; index `0` is the source instance and index `1` is the destination instance
- **name**: The instance name; the source references the input variable source_rds_name and the destination references the input variable dest_rds_name
- **flavor**: The instance flavor, assigned by referencing the input variable rds_flavor; when empty, the queried flavor is used automatically
- **security_group_id**: The ID of the security group to which the instance belongs, assigned by referencing the ID of the security group resource
- **subnet_id**: The ID of the subnet to which the instance belongs, assigned by referencing the ID of the subnet resource
- **vpc_id**: The ID of the VPC to which the instance belongs, assigned by referencing the ID of the VPC resource
- **fixed_ip**: The private IP of the instance; the source references the input variable source_rds_fixed_ip and the destination references the input variable dest_rds_fixed_ip
- **ha_replication_mode**: The primary/standby replication mode, set to `semisync`
- **availability_zone**: The availability zones of the instance, assigned by referencing the availability zones data source
- **db**: The database configuration, where **password** is assigned by referencing the input variable db_password
- **volume**: The storage configuration, with disk type `CLOUDSSD` and size `40` GB

### 6. Create DRS Migration Job

Add the following script in the TF file to create a DRS migration job:

```hcl
# Create a DRS migration job in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "job_name" {
  description = "The DRS job name"
  type        = string
}

variable "description" {
  description = "The description of the DRS job"
  type        = string
  default     = ""
}

resource "huaweicloud_drs_job" "test" {
  name           = var.job_name
  type           = "migration"
  engine_type    = "mysql"
  direction      = "up"
  net_type       = "eip"
  migration_type = "FULL_INCR_TRANS"
  description    = var.description
  force_destroy  = true

  source_db {
    engine_type = "mysql"
    ip          = huaweicloud_rds_instance.test[0].fixed_ip
    port        = 3306
    user        = "root"
    password    = var.db_password
    ssl_enabled = false
  }

  destination_db {
    region      = huaweicloud_rds_instance.test[1].region
    ip          = huaweicloud_rds_instance.test[1].fixed_ip
    port        = 3306
    engine_type = "mysql"
    user        = "root"
    password    = var.db_password
    instance_id = huaweicloud_rds_instance.test[1].id
    subnet_id   = huaweicloud_rds_instance.test[1].subnet_id
  }

  lifecycle {
    ignore_changes = [
      source_db.0.password, destination_db.0.password, force_destroy, action,
    ]
  }
}
```

**Parameter Description**:
- **name**: The job name, assigned by referencing the input variable job_name
- **type**: The job type, set to `migration`
- **engine_type**: The database engine type, set to `mysql`
- **direction**: The migration direction, set to `up` (to the cloud)
- **net_type**: The network type, set to `eip`
- **migration_type**: The migration mode, set to `FULL_INCR_TRANS` (full + incremental)
- **description**: The job description, assigned by referencing the input variable description
- **force_destroy**: Whether to force deletion, set to `true`
- **source_db**: The source database configuration, where **ip** references the private IP of the source RDS instance and **password** is assigned by referencing the input variable db_password
- **destination_db**: The destination database configuration, where **instance_id** references the ID of the destination RDS instance and **subnet_id** references the subnet ID of the destination RDS instance

### 7. Create LTS Log Group and Log Stream

Add the following script in the TF file to create an LTS log group and log stream:

```hcl
# Create an LTS log group and log stream in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "lts_group_name" {
  description = "The name of the LTS log group"
  type        = string
}

variable "lts_ttl_in_days" {
  description = "The log retention period in days"
  type        = number
  default     = 30
}

variable "lts_stream_name" {
  description = "The name of the LTS log stream"
  type        = string
}

resource "huaweicloud_lts_group" "test" {
  group_name  = var.lts_group_name
  ttl_in_days = var.lts_ttl_in_days
}

resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.lts_stream_name
  is_favorite = true
}
```

**Parameter Description**:
- **group_name**: The log group name, assigned by referencing the input variable lts_group_name
- **ttl_in_days**: The log retention period in days, assigned by referencing the input variable lts_ttl_in_days, defaulting to `30`
- **group_id**: The ID of the log group to which the log stream belongs, assigned by referencing the ID of the log group resource
- **stream_name**: The log stream name, assigned by referencing the input variable lts_stream_name
- **is_favorite**: Whether to favorite the log stream, set to `true`

### 8. Create DRS Job LTS Configuration

Add the following script in the TF file to deliver DRS job logs to LTS:

```hcl
# Create a DRS job LTS configuration in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_drs_lts_config" "test" {
  job_id        = huaweicloud_drs_job.test.id
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id
}
```

**Parameter Description**:
- **job_id**: The DRS job ID, assigned by referencing the ID of the DRS job resource
- **log_group_id**: The log group ID, assigned by referencing the ID of the LTS log group resource
- **log_stream_id**: The log stream ID, assigned by referencing the ID of the LTS log stream resource

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Network configuration
vpc_name            = "drs-vpc"
subnet_name         = "drs-subnet"
security_group_name = "drs-secgroup"

# RDS instance configuration
source_rds_name     = "drs-source-rds"
dest_rds_name       = "drs-dest-rds"
source_rds_fixed_ip = "192.168.0.10"
dest_rds_fixed_ip   = "192.168.0.11"
db_password         = "your-strong-password"

# DRS job configuration
job_name = "drs-migration-job"

# LTS configuration
lts_group_name  = "drs-lts-group"
lts_stream_name = "drs-lts-stream"
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

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DRS job LTS configuration
4. Run `terraform show` to view the created DRS job LTS configuration

## Reference Information

- [Huawei Cloud Data Replication Service Product Documentation](https://support.huaweicloud.com/drs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DRS LTS Configuration](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/lts-config)
