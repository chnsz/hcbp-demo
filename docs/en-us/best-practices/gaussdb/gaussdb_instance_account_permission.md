# Deploy GaussDB Instance Account Permission

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud, supporting both centralized and distributed deployment modes. In real-world business scenarios, database administrators need to grant different access permissions to different database accounts, for example, granting read-only permission to read-only accounts and read-write permission to business accounts, so as to ensure data security and follow the principle of least privilege.

This best practice will introduce how to use Terraform to automatically deploy GaussDB instance account permission, including VPC creation, subnet configuration, security group configuration, GaussDB instance creation, database creation, database account creation, schema creation, and database account permission setting.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [GaussDB Flavors (data.huaweicloud_gaussdb_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB Instance (huaweicloud_gaussdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB Database (huaweicloud_gaussdb_database)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_database)
- [GaussDB Instance Database Account (huaweicloud_gaussdb_instance_database_account)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance_database_account)
- [GaussDB Schema (huaweicloud_gaussdb_schema)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_schema)
- [GaussDB Instance Account Permission (huaweicloud_gaussdb_instance_account_permission)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance_account_permission)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_gaussdb_instance

data.huaweicloud_gaussdb_flavors
    └── huaweicloud_gaussdb_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_gaussdb_instance

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
    └── huaweicloud_gaussdb_instance

random_password
    └── huaweicloud_gaussdb_instance
    └── huaweicloud_gaussdb_instance_database_account

huaweicloud_gaussdb_instance
    └── huaweicloud_gaussdb_database
        └── huaweicloud_gaussdb_schema
        └── huaweicloud_gaussdb_instance_account_permission
    └── huaweicloud_gaussdb_instance_database_account
        └── huaweicloud_gaussdb_instance_account_permission
    └── huaweicloud_gaussdb_schema
        └── huaweicloud_gaussdb_instance_account_permission
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a Virtual Private Cloud

Add the following script to the TF file (such as main.tf):

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
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr, with a default value of `192.168.0.0/16`
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; when empty, the default enterprise project is used

### 3. Create a Virtual Private Cloud Subnet

Add the following script to the TF file (such as main.tf):

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
- **vpc_id**: Assigned by referencing the ID of the virtual private cloud resource
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; when empty, it is automatically calculated based on the VPC CIDR
- **gateway_ip**: Assigned by referencing the input variable gateway_ip; when empty, it is automatically calculated based on the subnet CIDR

### 4. Query Availability Zones and GaussDB Flavors

Add the following script to the TF file (such as main.tf):

```hcl
# Query availability zones and GaussDB flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_gaussdb_flavors" "test" {
  version = "8.201"
  ha_mode = "centralization_standard"
}
```

**Parameter Description**:
- **version**: GaussDB version, fixed to `8.201`
- **ha_mode**: High availability mode, fixed to `centralization_standard`

### 5. Create a Security Group and Security Group Rule

Add the following script to the TF file (such as main.tf):

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
- **name**: Assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to `true`
- **security_group_id**: Assigned by referencing the ID of the security group resource
- **direction**: Rule direction, set to `ingress`
- **ethertype**: IP protocol type, set to `IPv4`
- **protocol**: Protocol type, set to `tcp`
- **ports**: Assigned by referencing the input variable security_group_rule_ports, which opens the ports required by the GaussDB instance by default
- **remote_ip_prefix**: Assigned by referencing the CIDR of the VPC

### 6. Create a Random Password

Add the following script to the TF file (such as main.tf):

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
- **count**: Creates a random password when the input variable instance_password is empty
- **length**: Password length, set to `12`
- **special**: Whether to include special characters, set to `true`
- **override_special**: The set of allowed special characters
- **min_lower**, **min_upper**, **min_numeric**, **min_special**: The minimum number of characters of each type

### 7. Create a GaussDB Instance

Add the following script to the TF file (such as main.tf):

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
  flavor            = try(data.huaweicloud_gaussdb_flavors.test.flavors[0].spec_code, "")
  name              = var.instance_name
  password          = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  replica_num       = 3
  availability_zone = join(",", try(slice(data.huaweicloud_availability_zones.test.names, 0, 3), []))

  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  ha {
    mode             = "centralization_standard"
    replication_mode = "sync"
    consistency      = "strong"
    instance_mode    = "basic"
  }

  volume {
    type = var.instance_volume_type
    size = var.instance_volume_size
  }
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing the ID of the virtual private cloud resource
- **subnet_id**: Assigned by referencing the ID of the subnet resource
- **security_group_id**: Assigned by referencing the ID of the security group resource
- **flavor**: Assigned by referencing the spec code of the GaussDB flavors data source
- **name**: Assigned by referencing the input variable instance_name
- **password**: Uses the input variable instance_password when it is not empty, otherwise uses the random password
- **replica_num**: Number of replicas, set to `3`
- **availability_zone**: Assigned by referencing the first three availability zones of the availability zones data source
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id
- **ha.mode**: High availability mode, set to `centralization_standard`
- **ha.replication_mode**: Replication mode, set to `sync`
- **ha.consistency**: Consistency level, set to `strong`
- **ha.instance_mode**: Instance mode, set to `basic`
- **volume.type**: Assigned by referencing the input variable instance_volume_type
- **volume.size**: Assigned by referencing the input variable instance_volume_size

### 8. Create a GaussDB Database

Add the following script to the TF file (such as main.tf):

```hcl
# Create a GaussDB database resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "db_name" {
  description = "The database name"
  type        = string
}

variable "db_owner" {
  description = "The database owner"
  type        = string
}

resource "huaweicloud_gaussdb_database" "test" {
  instance_id   = huaweicloud_gaussdb_instance.test.id
  name          = var.db_name
  character_set = "UTF8"
  owner         = var.db_owner
  template      = "template0"
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the ID of the GaussDB instance resource
- **name**: Assigned by referencing the input variable db_name
- **character_set**: Character set, set to `UTF8`
- **owner**: Assigned by referencing the input variable db_owner
- **template**: Template database, set to `template0`

### 9. Create a GaussDB Instance Database Account

Add the following script to the TF file (such as main.tf):

```hcl
# Create a GaussDB instance database account resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "database_account_name" {
  description = "The name of the database account"
  type        = string
}

variable "database_account_password" {
  description = "The password of the database account"
  type        = string
  default     = ""
  sensitive   = true
}

variable "is_login_only" {
  description = "Whether the database account supports login only. The valid values are true and false"
  type        = string
  default     = "false"
}

resource "huaweicloud_gaussdb_instance_database_account" "test" {
  instance_id   = huaweicloud_gaussdb_instance.test.id
  name          = var.database_account_name
  password      = var.database_account_password != "" ? var.database_account_password : try(random_password.test[0].result, null)
  is_login_only = var.is_login_only
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the ID of the GaussDB instance resource
- **name**: Assigned by referencing the input variable database_account_name
- **password**: Uses the input variable database_account_password when it is not empty, otherwise uses the random password
- **is_login_only**: Assigned by referencing the input variable is_login_only, with a default value of `false`

### 10. Create a GaussDB Schema

Add the following script to the TF file (such as main.tf):

```hcl
# Create a GaussDB schema resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "schema_name" {
  description = "The schema name"
  type        = string
}

resource "huaweicloud_gaussdb_schema" "test" {
  instance_id = huaweicloud_gaussdb_instance.test.id
  db_name     = huaweicloud_gaussdb_database.test.name
  name        = var.schema_name
  owner       = var.db_owner
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the ID of the GaussDB instance resource
- **db_name**: Assigned by referencing the name of the GaussDB database resource
- **name**: Assigned by referencing the input variable schema_name
- **owner**: Assigned by referencing the input variable db_owner

### 11. Set GaussDB Instance Account Permission

Add the following script to the TF file (such as main.tf):

```hcl
# Set GaussDB instance account permission in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "permission_readonly" {
  description = "Whether the database account permission is read-only. The valid values are true and false"
  type        = string
  default     = "true"
}

resource "huaweicloud_gaussdb_instance_account_permission" "test" {
  instance_id = huaweicloud_gaussdb_instance.test.id
  db_name     = huaweicloud_gaussdb_database.test.name

  users {
    name        = var.database_account_name
    readonly    = var.permission_readonly
    schema_name = huaweicloud_gaussdb_schema.test.name
  }

  depends_on = [
    huaweicloud_gaussdb_database.test,
    huaweicloud_gaussdb_instance_database_account.test,
    huaweicloud_gaussdb_schema.test
  ]
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the ID of the GaussDB instance resource
- **db_name**: Assigned by referencing the name of the GaussDB database resource
- **users.name**: Assigned by referencing the input variable database_account_name
- **users.readonly**: Assigned by referencing the input variable permission_readonly; `true` means read-only permission and `false` means read-write permission
- **users.schema_name**: Assigned by referencing the name of the GaussDB schema resource
- **depends_on**: Ensures that the database, database account, and schema are created before the permission is set

### 12. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name              = "tf_test_vpc"
subnet_name           = "tf_test_subnet"
security_group_name   = "tf_test_security_group"
instance_name         = "tf_test_gaussdb_instance"
database_account_name = "tf_test_account_name"
db_name               = "tf_test_db"
db_owner              = "root"
schema_name           = "tf_test_schema"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 13. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB instance account permission
4. Run `terraform show` to view the created GaussDB instance account permission

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Instance Account Permission](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-instance-account-permission)
