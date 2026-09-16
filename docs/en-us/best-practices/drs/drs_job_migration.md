# Deploy Migration Job

## Application Scenario

Data Replication Service (DRS) is a one-stop data replication service provided by Huawei Cloud, supporting efficient data replication between on-cloud, off-cloud, and cross-cloud environments while ensuring business continuity. In scenarios such as database cloud migration and database migration, it is usually necessary to migrate data from a source database to a destination database in full plus incremental mode without interrupting the source business.

This best practice will introduce how to use Terraform to automatically deploy a DRS migration job, including the creation of a VPC, subnet, security group, source and destination RDS MySQL instances, and the configuration of the DRS migration job. Through this practice, you can quickly master how to orchestrate a DRS migration job with Terraform and lay a solid foundation for subsequent database migration and operation work.

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

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_rds_instance

data.huaweicloud_rds_flavors
    └── huaweicloud_rds_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_rds_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_rds_instance

huaweicloud_rds_instance
    └── huaweicloud_drs_job
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration introduction.

### 2. Create a Virtual Private Cloud

Add the following script in the TF file (such as main.tf) to create a virtual private cloud:

```hcl
# Create a virtual private cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr

### 3. Create a Virtual Private Cloud Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create a virtual private cloud subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC resource created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The subnet CIDR block, assigned by referencing the input variable subnet_cidr; when it is an empty string, the subnet is automatically divided based on the VPC CIDR block
- **gateway_ip**: The subnet gateway IP, assigned by referencing the input variable gateway_ip; when it is an empty string, the gateway IP is automatically calculated based on the subnet CIDR block

### 4. Create a Security Group and Its Rules

Add the following script in the TF file (such as main.tf) to create a security group and security group rules:

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
- **delete_default_rules**: Whether to delete the default rules of the security group, set to true to keep only custom rules
- **security_group_id**: The ID of the security group to which the security group rule belongs, referencing the ID of the security group resource created in the previous step
- **ethertype**: The network type, set to IPv4
- **remote_ip_prefix**: The remote IP address range, set to 192.168.0.0/16
- **protocol**: The protocol type, set to tcp
- **direction**: The rule direction, the first rule is ingress and the second rule is egress
- **ports**: The port range, the ingress rule opens port 3306 and the egress rule does not restrict the port

### 5. Query Availability Zones and RDS Flavors

Add the following script in the TF file (such as main.tf) to query the availability zone list and RDS flavor list:

```hcl
# Query the availability zone list in the current region
data "huaweicloud_availability_zones" "test" {}

# Query the RDS flavor list
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

data "huaweicloud_rds_flavors" "test" {
  db_type       = var.rds_db_type
  db_version    = var.rds_db_version
  instance_mode = var.rds_instance_mode
}
```

**Parameter Description**:
- **db_type**: The database type, assigned by referencing the input variable rds_db_type
- **db_version**: The database version, assigned by referencing the input variable rds_db_version
- **instance_mode**: The instance mode, assigned by referencing the input variable rds_instance_mode

### 6. Create Source and Destination RDS Instances

Add the following script in the TF file (such as main.tf) to create the source and destination RDS MySQL instances:

```hcl
# Create RDS instance resources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **count**: The number of instances, set to 2, corresponding to the source instance (index 0) and the destination instance (index 1)
- **name**: The instance name, the source references the input variable source_rds_name and the destination references the input variable dest_rds_name
- **flavor**: The instance flavor, assigned by referencing the input variable rds_flavor; when it is an empty string, the first flavor queried by the data source is used
- **security_group_id**: The ID of the security group to which the instance belongs, referencing the ID of the security group resource created in the previous step
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet resource created in the previous step
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC resource created in the previous step
- **fixed_ip**: The fixed IP address of the instance, the source references the input variable source_rds_fixed_ip and the destination references the input variable dest_rds_fixed_ip
- **ha_replication_mode**: The HA replication mode, set to semisync
- **availability_zone**: The availability zone list of the instance, referencing the query result of the availability zone data source
- **db**: The database configuration, including password (referencing the input variable db_password), type (MySQL), version (5.7) and port (3306)
- **volume**: The storage configuration, including storage type (CLOUDSSD) and storage size (40GB)

### 7. Create a DRS Migration Job

Add the following script in the TF file (such as main.tf) to create a DRS migration job:

```hcl
# Create a DRS job resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "job_name" {
  description = "The DRS job name"
  type        = string
}

variable "description" {
  description = "The description of the DRS job"
  type        = string
  default     = ""
}

variable "tags" {
  description = "The tags of the DRS job"
  type        = map(string)
  default     = {}
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

  tags = var.tags

  lifecycle {
    ignore_changes = [
      source_db.0.password, destination_db.0.password, force_destroy, action,
    ]
  }
}
```

**Parameter Description**:
- **name**: The DRS job name, assigned by referencing the input variable job_name
- **type**: The job type, set to migration
- **engine_type**: The database engine type, set to mysql
- **direction**: The migration direction, set to up
- **net_type**: The network type, set to eip
- **migration_type**: The migration mode, set to FULL_INCR_TRANS (full plus incremental)
- **description**: The job description, assigned by referencing the input variable description
- **force_destroy**: Whether to force destroy, set to true
- **source_db**: The source database configuration, including engine type (mysql), IP address (referencing the fixed IP of the source RDS instance), port (3306), user (root), password (referencing the input variable db_password) and SSL switch (false)
- **destination_db**: The destination database configuration, including region (referencing the region of the destination RDS instance), IP address (referencing the fixed IP of the destination RDS instance), port (3306), engine type (mysql), user (root), password (referencing the input variable db_password), instance ID (referencing the ID of the destination RDS instance) and subnet ID (referencing the subnet ID of the destination RDS instance)
- **tags**: The job tags, assigned by referencing the input variable tags
- **lifecycle**: The lifecycle configuration, ignoring changes to the source_db.0.password, destination_db.0.password, force_destroy and action attributes

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# VPC and network variables
vpc_name            = "your_vpc"
subnet_name         = "your_subnet"
security_group_name = "your_security_group"

# RDS instance variables
source_rds_name     = "your_source_rds"
dest_rds_name       = "your_dest_rds"
rds_flavor          = "rds.mysql.x1.large.2.ha"
source_rds_fixed_ip = "192.168.0.58"
dest_rds_fixed_ip   = "192.168.0.59"
db_password         = "TestDrs@123"

# DRS job variables
job_name = "your_drs_job"
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

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DRS migration job
4. Run `terraform show` to view the created DRS migration job

## Reference Information

- [Huawei Cloud Data Replication Service Product Documentation](https://support.huaweicloud.com/drs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DRS Migration Job](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/drs-job-migration)
