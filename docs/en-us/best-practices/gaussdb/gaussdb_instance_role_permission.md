# Deploy Instance Database Role Permission

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud, supporting both centralized and distributed deployment modes. In real-world business scenarios, database administrators need to grant different data access permissions to different database roles, for example, granting read-only permission to read-only roles and read-write permission to business roles, so as to ensure data access security and compliance.

This best practice will introduce how to use Terraform to automatically deploy GaussDB instance database role permission, including VPC creation, subnet configuration, security group configuration, GaussDB instance creation, database creation, database role creation, schema creation, and database role permission setting.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [GaussDB Flavors (data.huaweicloud_gaussdb_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB Instance (huaweicloud_gaussdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB Database (huaweicloud_gaussdb_database)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_database)
- [GaussDB Instance Database Role (huaweicloud_gaussdb_instance_database_role)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance_database_role)
- [GaussDB Schema (huaweicloud_gaussdb_schema)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_schema)
- [GaussDB Instance Role Permission (huaweicloud_gaussdb_instance_role_permission)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance_role_permission)

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
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_gaussdb_instance

huaweicloud_gaussdb_instance
    ├── huaweicloud_gaussdb_database
    │   └── huaweicloud_gaussdb_schema
    ├── huaweicloud_gaussdb_instance_database_role
    └── huaweicloud_gaussdb_instance_role_permission
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a VPC and Subnet

Add the following script in the TF file (such as main.tf) to create a VPC and subnet:

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

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
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
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
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
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id; the default enterprise project is used when it is empty
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC created above
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet, assigned by referencing the input variable subnet_cidr; it is automatically calculated based on the VPC CIDR when empty
- **gateway_ip**: The gateway IP address of the subnet, assigned by referencing the input variable gateway_ip; it is automatically calculated based on the subnet CIDR when empty

### 3. Query Availability Zones and GaussDB Flavors

Add the following script in the TF file (such as main.tf) to query availability zones and GaussDB flavors:

```hcl
# Query availability zones and GaussDB flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_gaussdb_flavors" "test" {
  version = "8.201"
  ha_mode = "centralization_standard"
}
```

**Parameter Description**:
- **version**: The GaussDB database version, set to 8.201 here
- **ha_mode**: The instance deployment mode, set to centralized standard (centralization_standard) here

### 4. Create a Security Group and Rule

Add the following script in the TF file (such as main.tf) to create a security group and its inbound rule:

```hcl
# Create security group and rule resources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **delete_default_rules**: Whether to delete the default rules of the security group, set to true here
- **security_group_id**: The ID of the security group to which the rule belongs, assigned by referencing the ID of the security group created above
- **direction**: The rule direction, set to inbound (ingress) here
- **ethertype**: The IP protocol type, set to IPv4 here
- **protocol**: The protocol type, set to TCP here
- **ports**: The rule port range, assigned by referencing the input variable security_group_rule_ports
- **remote_ip_prefix**: The remote IP address range, assigned by referencing the CIDR block of the VPC

### 5. Create a Random Password

Add the following script in the TF file (such as main.tf) to automatically generate a random password when the instance password is not specified:

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
- **count**: The random password is created when the input variable instance_password is empty; otherwise it is not created
- **length**: The password length, set to 12 here
- **special**: Whether to include special characters, set to true here
- **override_special**: The set of allowed special characters
- **min_lower**, **min_upper**, **min_numeric**, **min_special**: The minimum number of lowercase letters, uppercase letters, digits, and special characters in the password

### 6. Create a GaussDB Instance

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
- **vpc_id**: The ID of the VPC to which the instance belongs, assigned by referencing the ID of the VPC created above
- **subnet_id**: The ID of the subnet to which the instance belongs, assigned by referencing the ID of the subnet created above
- **security_group_id**: The ID of the security group to which the instance belongs, assigned by referencing the ID of the security group created above
- **flavor**: The instance flavor, assigned by referencing the first flavor of the GaussDB flavors data source
- **name**: The instance name, assigned by referencing the input variable instance_name
- **password**: The instance password, assigned by referencing the input variable instance_password; the random password is used when it is empty
- **replica_num**: The number of replicas, set to 3 here
- **availability_zone**: The availability zones, assigned by referencing the first three availability zones of the availability zones data source
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **ha.mode**: The instance deployment mode, set to centralized standard (centralization_standard) here
- **ha.replication_mode**: The replication mode, set to synchronous (sync) here
- **ha.consistency**: The consistency, set to strong consistency (strong) here
- **ha.instance_mode**: The instance mode, set to basic here
- **volume.type**: The storage type, assigned by referencing the input variable instance_volume_type
- **volume.size**: The storage size, assigned by referencing the input variable instance_volume_size

### 7. Create a Database

Add the following script in the TF file (such as main.tf) to create a database:

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
- **instance_id**: The ID of the GaussDB instance to which the database belongs, assigned by referencing the ID of the instance created above
- **name**: The database name, assigned by referencing the input variable db_name
- **character_set**: The database character set, set to UTF8 here
- **owner**: The database owner, assigned by referencing the input variable db_owner
- **template**: The database template, set to template0 here

### 8. Create a Database Role

Add the following script in the TF file (such as main.tf) to create a database role:

```hcl
# Create a GaussDB instance database role resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "database_role_name" {
  description = "The name of the database role"
  type        = string
}

variable "database_role_password" {
  description = "The password of the database role"
  type        = string
  default     = ""
  sensitive   = true
}

resource "huaweicloud_gaussdb_instance_database_role" "test" {
  instance_id = huaweicloud_gaussdb_instance.test.id
  name        = var.database_role_name
  password    = (var.database_role_password != "" ? var.database_role_password :
  try(random_password.test[0].result, null))
}
```

**Parameter Description**:
- **instance_id**: The ID of the GaussDB instance to which the database role belongs, assigned by referencing the ID of the instance created above
- **name**: The database role name, assigned by referencing the input variable database_role_name
- **password**: The database role password, assigned by referencing the input variable database_role_password; the random password is used when it is empty

### 9. Create a Schema

Add the following script in the TF file (such as main.tf) to create a schema:

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
- **instance_id**: The ID of the GaussDB instance to which the schema belongs, assigned by referencing the ID of the instance created above
- **db_name**: The name of the database to which the schema belongs, assigned by referencing the name of the database created above
- **name**: The schema name, assigned by referencing the input variable schema_name
- **owner**: The schema owner, assigned by referencing the input variable db_owner

### 10. Set Database Role Permission

Add the following script in the TF file (such as main.tf) to set the database role permission:

```hcl
# Create a GaussDB instance role permission resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "permission_readonly" {
  description = "Whether the database role permission is read-only. The valid values are true and false"
  type        = string
  default     = "true"
}

resource "huaweicloud_gaussdb_instance_role_permission" "test" {
  instance_id = huaweicloud_gaussdb_instance.test.id
  db_name     = huaweicloud_gaussdb_database.test.name

  user {
    name     = var.database_role_name
    readonly = var.permission_readonly
    schema   = huaweicloud_gaussdb_schema.test.name
  }

  depends_on = [
    huaweicloud_gaussdb_database.test,
    huaweicloud_gaussdb_instance_database_role.test,
    huaweicloud_gaussdb_schema.test
  ]
}
```

**Parameter Description**:
- **instance_id**: The GaussDB instance ID, assigned by referencing the ID of the instance created above
- **db_name**: The database name, assigned by referencing the name of the database created above
- **user.name**: The database role name, assigned by referencing the input variable database_role_name
- **user.readonly**: Whether the database role permission is read-only, assigned by referencing the input variable permission_readonly; true means read-only permission and false means read-write permission
- **user.schema**: The schema name, assigned by referencing the name of the schema created above
- **depends_on**: The explicit dependency, ensuring that the database, database role, and schema are created before the permission is set

### 11. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

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
database_role_name  = "tf_test_role_name"
db_name             = "tf_test_db"
db_owner            = "root"
schema_name         = "tf_test_schema"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB instance database role permission
4. Run `terraform show` to view the created GaussDB instance database role permission

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Instance Database Role Permission](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-instance-role-permission)
