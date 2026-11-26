# Deploy Two-Way Synchronization Jobs

## Application Scenario

Data Replication Service (DRS) is a one-stop data replication service provided by Huawei Cloud, supporting real-time synchronization between databases while ensuring business continuity. In scenarios such as active-active business, read/write splitting, and data aggregation, bidirectional synchronization links are often required between two database instances so that data on both ends stays consistent.

This best practice will introduce how to use Terraform to automatically deploy two DRS two-way synchronization jobs, including the creation of a VPC, subnet, and security group, two RDS MySQL instances and the databases to be synchronized on them, as well as the creation, object selection, and start of the forward (up) and reverse (down) DRS synchronization jobs.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DRS Node Types (data.huaweicloud_drs_node_types)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/drs_node_types)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RDS Instance (huaweicloud_rds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [RDS MySQL Database (huaweicloud_rds_mysql_database)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_mysql_database)
- [DRS Job (huaweicloud_drs_job)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_job)
- [DRS Batch Select Objects (huaweicloud_drs_batch_select_objects)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_batch_select_objects)
- [DRS Start Job (huaweicloud_drs_start_job)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_start_job)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_rds_instance
        └── huaweicloud_rds_mysql_database
            └── huaweicloud_drs_batch_select_objects
                └── huaweicloud_drs_start_job

data.huaweicloud_drs_node_types
    └── huaweicloud_drs_job
        └── huaweicloud_drs_batch_select_objects
            └── huaweicloud_drs_start_job

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_rds_instance

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
        └── huaweicloud_rds_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

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
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr
- **vpc_id**: The VPC to which the subnet belongs, assigned by referencing the ID of the VPC created above
- **cidr**: The CIDR block of the subnet; when the input variable subnet_cidr is empty, the subnet is automatically divided based on the VPC CIDR
- **gateway_ip**: The gateway IP of the subnet; when the input variable gateway_ip is empty, the gateway address is automatically calculated based on the subnet CIDR

### 3. Create a Security Group and Its Rules

Add the following script in the TF file (such as main.tf):

```hcl
# Create security group and its rules in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **delete_default_rules**: Whether to delete the default rules of the security group; set to true to keep only custom rules
- **security_group_id**: The security group to which the rule belongs, assigned by referencing the ID of the security group created above
- **direction**: The rule direction; the first is ingress and the second is egress
- **ports**: The ingress rule opens port 3306 for MySQL database access
- **remote_ip_prefix**: The remote CIDR block to which the rule applies, set to 192.168.0.0/16

### 4. Query the Availability Zones

Add the following script in the TF file (such as main.tf):

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- This data source requires no additional parameters and is used to obtain the availability zone list in the current region for the RDS instances to select primary and standby availability zones

### 5. Create Two RDS MySQL Instances

Add the following script in the TF file (such as main.tf):

```hcl
# Create two RDS MySQL instances in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "rds1_name" {
  description = "The name of the first RDS instance"
  type        = string
}

variable "rds2_name" {
  description = "The name of the second RDS instance"
  type        = string
}

variable "rds_flavor" {
  description = "The flavor of the RDS instances"
  type        = string
  default     = "rds.mysql.x1.large.2.ha"
}

variable "rds1_fixed_ip" {
  description = "The fixed IP address of the first RDS instance"
  type        = string
  default     = "192.168.0.50"
}

variable "rds2_fixed_ip" {
  description = "The fixed IP address of the second RDS instance"
  type        = string
  default     = "192.168.0.51"
}

variable "db_password" {
  description = "The password for the RDS root user and DRS database connections"
  type        = string
  sensitive   = true
}

resource "huaweicloud_rds_instance" "test" {
  count = 2

  depends_on = [
    huaweicloud_networking_secgroup_rule.test,
  ]

  name                = count.index == 0 ? var.rds1_name : var.rds2_name
  flavor              = var.rds_flavor
  security_group_id   = huaweicloud_networking_secgroup.test.id
  subnet_id           = huaweicloud_vpc_subnet.test.id
  vpc_id              = huaweicloud_vpc.test.id
  fixed_ip            = count.index == 0 ? var.rds1_fixed_ip : var.rds2_fixed_ip
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
- **count**: Creates two RDS instances; index 0 is the first instance and index 1 is the second instance
- **name**: The instance name; index 0 references the input variable rds1_name and index 1 references the input variable rds2_name
- **flavor**: The instance flavor, assigned by referencing the input variable rds_flavor
- **security_group_id**: The security group bound to the instance, assigned by referencing the ID of the security group created above
- **subnet_id**: The subnet to which the instance belongs, assigned by referencing the ID of the subnet created above
- **vpc_id**: The VPC to which the instance belongs, assigned by referencing the ID of the VPC created above
- **fixed_ip**: The fixed IP of the instance; index 0 references the input variable rds1_fixed_ip and index 1 references the input variable rds2_fixed_ip, both of which must be within the subnet CIDR
- **ha_replication_mode**: The replication mode of the HA instance, set to semisync
- **availability_zone**: The primary and standby availability zones of the instance, assigned by referencing the availability zone names returned by the data source
- **db.password**: The password of the database root user, assigned by referencing the input variable db_password
- **db.type**: The database engine type, set to MySQL
- **db.version**: The database version, set to 5.7
- **db.port**: The database port, set to 3306
- **volume.type**: The storage type, set to CLOUDSSD
- **volume.size**: The storage size, set to 40GB

### 6. Create the Databases to Be Synchronized on Both Instances

Add the following script in the TF file (such as main.tf):

```hcl
# Create RDS MySQL databases in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "db_name" {
  description = "The name of the database to be synchronized in both directions"
  type        = string
  default     = "test"
}

resource "huaweicloud_rds_mysql_database" "test" {
  count = 2

  instance_id   = huaweicloud_rds_instance.test[count.index].id
  name          = var.db_name
  character_set = "utf8"
}
```

**Parameter Description**:
- **count**: Creates a database on each of the two RDS instances so that both ends have data that can serve as the source database
- **instance_id**: The RDS instance to which the database belongs, assigned by referencing the ID of the RDS instance created above
- **name**: The database name, assigned by referencing the input variable db_name
- **character_set**: The database character set, set to utf8

### 7. Query the DRS Job Node Types

Add the following script in the TF file (such as main.tf):

```hcl
# Query the DRS node types in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_drs_node_types" "test" {
  count = 2

  engine_type = "mysql"
  type        = "sync"
  direction   = count.index == 0 ? "up" : "down"
}
```

**Parameter Description**:
- **count**: Queries the node types for the forward (up) and reverse (down) data directions respectively
- **engine_type**: The database engine type, set to mysql
- **type**: The job type, set to sync (synchronization job)
- **direction**: The data direction; index 0 is up (forward) and index 1 is down (reverse)

### 8. Create Two DRS Synchronization Jobs

Add the following script in the TF file (such as main.tf):

```hcl
# Create two DRS synchronization jobs in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "forward_job_name" {
  description = "The name of the forward DRS job (from the first RDS instance to the second one)"
  type        = string
}

variable "reverse_job_name" {
  description = "The name of the reverse DRS job (from the second RDS instance to the first one)"
  type        = string
}

variable "forward_job_description" {
  description = "The description of the forward DRS job"
  type        = string
  default     = ""
}

variable "reverse_job_description" {
  description = "The description of the reverse DRS job"
  type        = string
  default     = ""
}

variable "db_user" {
  description = "The database user name used by the RDS instances and DRS jobs"
  type        = string
  default     = "root"
}

variable "limit_speed" {
  description = "The speed limit of the DRS jobs. If not specified, the jobs are not limited"
  type = object({
    speed      = string
    start_time = string
    end_time   = string
  })
  default = null
}

resource "huaweicloud_drs_job" "test" {
  count = 2

  name                    = count.index == 0 ? var.forward_job_name : var.reverse_job_name
  type                    = "sync"
  engine_type             = "mysql"
  direction               = count.index == 0 ? "up" : "down"
  node_type               = try(data.huaweicloud_drs_node_types.test[count.index].node_types[0], null)
  net_type                = "vpc"
  migration_type          = "FULL_INCR_TRANS"
  description             = count.index == 0 ? var.forward_job_description : var.reverse_job_description
  force_destroy           = true
  is_start_job            = false
  is_pre_check            = false
  destination_db_readnoly = false
  charging_mode           = "postPaid"

  source_db {
    engine_type = "mysql"
    ip          = huaweicloud_rds_instance.test[count.index].fixed_ip
    port        = 3306
    user        = var.db_user
    password    = var.db_password
    vpc_id      = count.index == 0 ? huaweicloud_rds_instance.test[count.index].vpc_id : null
    instance_id = count.index == 1 ? huaweicloud_rds_instance.test[count.index].id : null
    subnet_id   = huaweicloud_rds_instance.test[count.index].subnet_id
  }

  destination_db {
    region      = count.index == 0 ? huaweicloud_rds_instance.test[1 - count.index].region : null
    ip          = huaweicloud_rds_instance.test[1 - count.index].fixed_ip
    port        = 3306
    engine_type = "mysql"
    user        = var.db_user
    password    = var.db_password
    instance_id = count.index == 0 ? huaweicloud_rds_instance.test[1 - count.index].id : null
    vpc_id      = count.index == 1 ? huaweicloud_rds_instance.test[1 - count.index].vpc_id : null
    subnet_id   = huaweicloud_rds_instance.test[1 - count.index].subnet_id
  }

  policy_config {
    filter_ddl_policy = "drop_database"
    conflict_policy   = "overwrite"
    index_trans       = true
  }

  dynamic "limit_speed" {
    for_each = var.limit_speed == null ? [] : [var.limit_speed]

    content {
      speed      = limit_speed.value.speed
      start_time = limit_speed.value.start_time
      end_time   = limit_speed.value.end_time
    }
  }

  lifecycle {
    ignore_changes = [
      source_db.0.password, destination_db.0.password, force_destroy,
    ]
  }
}
```

**Parameter Description**:
- **count**: Creates two DRS jobs; index 0 is the forward job (from the first instance to the second) and index 1 is the reverse job (from the second instance to the first)
- **name**: The job name; index 0 references the input variable forward_job_name and index 1 references the input variable reverse_job_name
- **type**: The job type, set to sync (synchronization job)
- **engine_type**: The database engine type, set to mysql
- **direction**: The data direction; index 0 is up and index 1 is down
- **node_type**: The job node type, assigned by referencing the node type returned by the DRS node types data source
- **net_type**: The network type, set to vpc
- **migration_type**: The migration type, set to FULL_INCR_TRANS (full plus incremental)
- **description**: The job description; index 0 references the input variable forward_job_description and index 1 references the input variable reverse_job_description
- **is_start_job**: Whether to start the job immediately after creation; set to false so that synchronization objects can be selected while the job is in the configuration status
- **is_pre_check**: Whether to perform a pre-check, set to false
- **charging_mode**: The charging mode, set to postPaid (pay-per-use)
- **source_db**: The source database information; index 0 uses the first instance as the source and index 1 uses the second instance as the source, including engine type, IP, port, username, password, VPC, instance ID, and subnet
- **destination_db**: The destination database information; index 0 uses the second instance as the destination and index 1 uses the first instance as the destination, including region, IP, port, engine type, username, password, instance ID, VPC, and subnet
- **policy_config**: The synchronization policy configuration, including the DDL filtering policy, conflict policy, and index transmission switch
- **limit_speed**: The speed limit configuration, which takes effect when the input variable limit_speed is not empty

### 9. Select Synchronization Objects for the Jobs

Add the following script in the TF file (such as main.tf):

```hcl
# Select synchronization objects for the DRS jobs in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_drs_batch_select_objects" "test" {
  count = 2

  jobs {
    job_id        = huaweicloud_drs_job.test[count.index].id
    selected      = true
    sync_database = true

    job {
      id          = huaweicloud_rds_mysql_database.test[count.index].name
      object_type = "database"
      object_name = huaweicloud_rds_mysql_database.test[count.index].name
      select      = "true"
    }
  }
}
```

**Parameter Description**:
- **count**: Selects synchronization objects for the forward and reverse jobs respectively
- **job_id**: The DRS job for which objects are selected, assigned by referencing the ID of the DRS job created above
- **selected**: Whether objects have been selected, set to true
- **sync_database**: Whether to synchronize the database, set to true
- **job.id**: The synchronization object ID, assigned by referencing the name of the database created above
- **job.object_type**: The object type, set to database
- **job.object_name**: The object name, assigned by referencing the name of the database created above
- **job.select**: Whether to select the object, set to true

### 10. Start the DRS Synchronization Jobs

Add the following script in the TF file (such as main.tf):

```hcl
# Start the DRS synchronization jobs in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_drs_start_job" "test" {
  count = 2

  depends_on = [huaweicloud_drs_batch_select_objects.test]

  job_id = huaweicloud_drs_job.test[count.index].id
}
```

**Parameter Description**:
- **count**: Starts the forward and reverse DRS jobs respectively
- **depends_on**: Ensures the jobs are started only after the synchronization objects have been selected
- **job_id**: The DRS job to be started, assigned by referencing the ID of the DRS job created above

### 11. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
vpc_name            = "tf_test_drs_vpc"
subnet_name         = "tf_test_drs_subnet"
security_group_name = "tf_test_drs_sg"
rds1_name           = "tf_test_drs_rds1"
rds2_name           = "tf_test_drs_rds2"
rds_flavor          = "rds.mysql.x1.large.2.ha"
rds1_fixed_ip       = "192.168.0.50"
rds2_fixed_ip       = "192.168.0.51"
db_password         = "<your_db_password>"
db_name             = "test"
forward_job_name    = "tf_test_forward_drs_job"
reverse_job_name    = "tf_test_reverse_drs_job"
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

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the two-way synchronization jobs
4. Run `terraform show` to view the created two-way synchronization jobs

## Reference Information

- [Huawei Cloud Data Replication Service Product Documentation](https://support.huaweicloud.com/drs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DRS Two-Way Synchronization Jobs](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/drs-job-two-way-synchronization)
