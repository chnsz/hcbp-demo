# Deploy GaussDB Database

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud, supporting both centralized and distributed deployment modes. In real-world scenarios, users often need to create dedicated business databases within a GaussDB OpenGauss instance and specify attributes such as character set, owner, and collation to meet the compatibility and data isolation requirements of different applications.

This best practice will introduce how to use Terraform to automatically deploy a GaussDB database, including VPC creation, subnet configuration, security group and security group rule configuration, availability zone query, GaussDB instance creation, and database creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB Instance (huaweicloud_gaussdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB Database (huaweicloud_gaussdb_database)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_database)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_gaussdb_instance.test
        └── huaweicloud_gaussdb_database.test

huaweicloud_vpc.test
    ├── huaweicloud_vpc_subnet.test
    │   └── huaweicloud_gaussdb_instance.test
    └── huaweicloud_networking_secgroup_rule.test

huaweicloud_networking_secgroup.test
    ├── huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_gaussdb_instance.test

random_password.test
    └── huaweicloud_gaussdb_instance.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

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

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr
- **enterprise_project_id**: The enterprise project ID, referencing the input variable enterprise_project_id when it is not empty, otherwise null

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
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet, automatically calculated from the VPC CIDR when the input variable subnet_cidr is empty, otherwise referencing its value
- **gateway_ip**: The gateway IP address of the subnet, automatically calculated from the subnet CIDR when the input variable gateway_ip is empty, otherwise referencing its value

### 4. Query the Availability Zone List

Add the following script in the TF file (such as main.tf) to query the availability zone list:

```hcl
# Query the availability zone list data source in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}
```

### 5. Create a Security Group and Security Group Rule

Add the following script in the TF file (such as main.tf) to create a security group and security group rule:

```hcl
# Create a security group and security group rule resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

variable "security_group_rule_ports" {
  description = "The security group ingress rule ports"
  type        = string
  default     = "2379-2380,5000-5001,5432-5532,6000,6500,12016,20050"
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  ports             = var.security_group_rule_ports
  remote_ip_prefix  = huaweicloud_vpc.test.cidr
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group, set to true
- **security_group_id**: The ID of the security group to which the rule belongs, referencing the ID of the security group created in the previous step
- **direction**: The rule direction, set to ingress
- **ethertype**: The IP protocol type, set to IPv4
- **protocol**: The protocol type, set to tcp
- **ports**: The rule port range, assigned by referencing the input variable security_group_rule_ports, which must include the system ports and database ports required for the GaussDB instance to run
- **remote_ip_prefix**: The remote IP prefix, referencing the CIDR block of the VPC

### 6. Create a Random Password

Add the following script in the TF file (such as main.tf) to automatically generate a random password when no instance password is specified:

```hcl
# Create a random password resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_password" {
  description = "The password for the GaussDB instance"
  type        = string
  default     = ""
  sensitive   = true
}

resource "random_password" "test" {
  count = var.instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "!@%^*-_=+"
  min_lower        = 1
  min_upper        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**Parameter Description**:
- **count**: Creates one random password when the input variable instance_password is empty, otherwise creates none
- **length**: The password length, set to 12
- **special**: Whether to include special characters, set to true
- **override_special**: The set of allowed special characters
- **min_lower**, **min_upper**, **min_numeric**, **min_special**: The minimum number of lowercase letters, uppercase letters, digits, and special characters respectively

### 7. Create a GaussDB Instance

Add the following script in the TF file (such as main.tf) to create a GaussDB instance:

```hcl
# Create a GaussDB instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The GaussDB instance name"
  type        = string
}

variable "instance_volume_type" {
  description = "The storage volume type"
  type        = string
  default     = "ULTRAHIGH"
}

variable "instance_volume_size" {
  description = "The storage volume size in GB"
  type        = number
  default     = 40
}

resource "huaweicloud_gaussdb_instance" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  flavor            = "gaussdb.opengauss.ee.dn.m6.2xlarge.8.in"
  name              = var.instance_name
  password          = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  sharding_num      = 1
  coordinator_num   = 2
  replica_num       = 3
  availability_zone = join(",", try(slice(data.huaweicloud_availability_zones.test.names, 0, 3), []))

  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  ha {
    mode             = "enterprise"
    replication_mode = "sync"
    consistency      = "strong"
  }

  volume {
    type = var.instance_volume_type
    size = var.instance_volume_size
  }
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC created in the previous step
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet created in the previous step
- **security_group_id**: The ID of the security group to which the instance belongs, referencing the ID of the security group created in the previous step
- **flavor**: The instance flavor, set to `gaussdb.opengauss.ee.dn.m6.2xlarge.8.in`
- **name**: The instance name, assigned by referencing the input variable instance_name
- **password**: The instance password, referencing the input variable instance_password when it is not empty, otherwise referencing the result generated by the random password resource
- **sharding_num**: The number of shards, set to 1
- **coordinator_num**: The number of coordinator nodes, set to 2
- **replica_num**: The number of replicas, set to 3
- **availability_zone**: The availability zones, taking the first three from the availability zone data source and joining them with commas
- **enterprise_project_id**: The enterprise project ID, referencing the input variable enterprise_project_id when it is not empty, otherwise null
- **ha.mode**: The high availability mode, set to enterprise
- **ha.replication_mode**: The replication mode, set to sync
- **ha.consistency**: The consistency level, set to strong
- **volume.type**: The storage type, assigned by referencing the input variable instance_volume_type
- **volume.size**: The storage size, assigned by referencing the input variable instance_volume_size

### 8. Create a GaussDB Database

Add the following script in the TF file (such as main.tf) to create a database in the GaussDB instance:

```hcl
# Create a GaussDB database resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "database_name" {
  description = "The name of the database"
  type        = string
}

variable "character_set" {
  description = "The database character set"
  type        = string
  default     = "UTF8"
}

variable "owner" {
  description = "The database user"
  type        = string
  default     = "root"
}

variable "template" {
  description = "The name of the database template"
  type        = string
  default     = ""
}

variable "lc_collate" {
  description = "The database collation"
  type        = string
  default     = "C"
}

variable "lc_ctype" {
  description = "The database classification"
  type        = string
  default     = "C"
}

resource "huaweicloud_gaussdb_database" "test" {
  instance_id   = huaweicloud_gaussdb_instance.test.id
  name          = var.database_name
  character_set = var.character_set
  owner         = var.owner
  template      = var.template
  lc_collate    = var.lc_collate
  lc_ctype      = var.lc_ctype
}
```

**Parameter Description**:
- **instance_id**: The ID of the GaussDB instance to which the database belongs, referencing the ID of the GaussDB instance created in the previous step
- **name**: The database name, assigned by referencing the input variable database_name; it must be 1-63 characters, cannot start with pg or a digit, and cannot be a template database name
- **character_set**: The database character set, assigned by referencing the input variable character_set, defaulting to UTF8
- **owner**: The database owner, assigned by referencing the input variable owner, defaulting to root, and must not be a system user (rdsAdmin, rdsMetric, rdsBackup, rdsRepl)
- **template**: The database template name, assigned by referencing the input variable template, which can be set to template0
- **lc_collate**: The database collation, assigned by referencing the input variable lc_collate, defaulting to C
- **lc_ctype**: The database character classification, assigned by referencing the input variable lc_ctype, defaulting to C

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
instance_name       = "tf_test_gaussdb_instance"
database_name       = "tf_test_db"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB database
4. Run `terraform show` to view the created GaussDB database

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Database](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-database)
