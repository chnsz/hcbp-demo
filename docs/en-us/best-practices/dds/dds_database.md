# Deploy Database Role and User

## Application Scenario

Document Database Service (DDS) is a high-performance, highly reliable, and secure distributed document database service provided by Huawei Cloud, fully compatible with the MongoDB protocol. In real-world business scenarios, in addition to creating a DDS instance, you also need to create database roles and users for the database to achieve fine-grained access control and permission management.

This best practice will introduce how to use Terraform to automatically create the database role and user of a DDS instance, including the creation of VPC, subnet, security group, DDS instance, database role, and database user, helping you quickly complete the initial configuration of the DDS database.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DDS Instance (huaweicloud_dds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)
- [DDS Database Role (huaweicloud_dds_database_role)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_database_role)
- [DDS Database User (huaweicloud_dds_database_user)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_database_user)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dds_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dds_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dds_instance

random_password
    └── huaweicloud_dds_database_user

huaweicloud_dds_instance
    ├── huaweicloud_dds_database_role
    └── huaweicloud_dds_database_user
        └── huaweicloud_dds_database_role
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones available for the DDS instance:

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone to which the DDS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable availability_zone is empty, otherwise it is not created

### 3. Create a VPC

Add the following script in the TF file (such as main.tf) to create a VPC:

```hcl
# Create a VPC in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
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
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr

### 4. Create a Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create a subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing the id of the resource huaweicloud_vpc.test
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: When the input variable subnet_cidr is empty, the subnet CIDR is automatically calculated based on the VPC CIDR, otherwise it is assigned by referencing the input variable subnet_cidr
- **gateway_ip**: When the input variable subnet_gateway_ip is empty, the gateway IP is automatically calculated based on the subnet CIDR, otherwise it is assigned by referencing the input variable subnet_gateway_ip

### 5. Create a Security Group

Add the following script in the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable security_group_name
- **delete_default_rules**: Set to true to delete the default rules of the security group

### 6. Create a Random Password

Add the following script in the TF file (such as main.tf) to automatically generate a random password when the instance password is not specified:

```hcl
# Create a random password in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_password" {
  description = "The DDS instance access password"
  type        = string
  sensitive   = true
  default     = ""
}

resource "random_password" "test" {
  count = var.instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "!@#%^*-_+?"
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**Parameter Description**:
- **count**: The resource is created when the input variable instance_password is empty, otherwise it is not created
- **length**: The length of the random password
- **special**: Whether to include special characters
- **override_special**: The set of allowed special characters
- **min_upper**: The minimum number of uppercase letters
- **min_lower**: The minimum number of lowercase letters
- **min_numeric**: The minimum number of digits
- **min_special**: The minimum number of special characters

### 7. Create a DDS Instance

Add the following script in the TF file (such as main.tf) to create a DDS instance:

```hcl
# Create a DDS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the DDS instance"
  type        = string
}

variable "instance_mode" {
  description = "The type of the DDS instance"
  type        = string
  default     = "ReplicaSet"
}

variable "database_type" {
  description = "The database version type of the DDS instance"
  type        = string
  default     = "DDS-Community"
}

variable "database_version" {
  description = "The database version of the DDS instance"
  type        = string
  default     = "4.0"
}

variable "storage_engine" {
  description = "The storage engine of the DDS instance"
  type        = string
  default     = "wiredTiger"
}

variable "node_type" {
  description = "The type of the DDS instance node"
  type        = string
  default     = "replica"
}

variable "node_number" {
  description = "The number of nodes of the DDS instance"
  type        = number
  default     = 3
}

variable "node_spec_code" {
  description = "The spec code of the DDS instance node"
  type        = string
  default     = "dds.mongodb.s6.large.2.repset"
  nullable    = false
}

variable "node_storage_type" {
  description = "The storage type of the DDS instance node"
  type        = string
  default     = "ULTRAHIGH"
}

variable "node_size" {
  description = "The disk size of the node of the DDS instance"
  type        = number
  default     = 10
}

variable "node_list" {
  description = "The node IDs to be deleted of the DDS instance"
  type        = list(string)
  default     = null
}

resource "huaweicloud_dds_instance" "test" {
  name              = var.instance_name
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  mode              = var.instance_mode
  password          = var.instance_password

  datastore {
    type           = var.database_type
    version        = var.database_version
    storage_engine = var.storage_engine
  }

  flavor {
    type      = var.node_type
    num       = var.node_number
    spec_code = var.node_spec_code
    storage   = var.node_storage_type
    size      = var.node_size
    node_list = var.node_list
  }
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable instance_name
- **availability_zone**: When the input variable availability_zone is empty, it references the first availability zone queried by the data source, otherwise it is assigned by referencing the input variable availability_zone
- **vpc_id**: Assigned by referencing the id of the resource huaweicloud_vpc.test
- **subnet_id**: Assigned by referencing the id of the resource huaweicloud_vpc_subnet.test
- **security_group_id**: Assigned by referencing the id of the resource huaweicloud_networking_secgroup.test
- **mode**: Assigned by referencing the input variable instance_mode
- **password**: Assigned by referencing the input variable instance_password
- **datastore.type**: Assigned by referencing the input variable database_type
- **datastore.version**: Assigned by referencing the input variable database_version
- **datastore.storage_engine**: Assigned by referencing the input variable storage_engine
- **flavor.type**: Assigned by referencing the input variable node_type
- **flavor.num**: Assigned by referencing the input variable node_number
- **flavor.spec_code**: Assigned by referencing the input variable node_spec_code
- **flavor.storage**: Assigned by referencing the input variable node_storage_type
- **flavor.size**: Assigned by referencing the input variable node_size
- **flavor.node_list**: Assigned by referencing the input variable node_list

### 8. Create a DDS Database Role

Add the following script in the TF file (such as main.tf) to create a DDS database role:

```hcl
# Create a DDS database role in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "database_role_name" {
  description = "The database role name"
  type        = string
}

resource "huaweicloud_dds_database_role" "test" {
  instance_id = huaweicloud_dds_instance.test.id
  name        = var.database_role_name
  db_name     = "admin"
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the id of the resource huaweicloud_dds_instance.test
- **name**: Assigned by referencing the input variable database_role_name
- **db_name**: The name of the database to which the database role belongs, fixed to admin

### 9. Create a DDS Database User

Add the following script in the TF file (such as main.tf) to create a DDS database user:

```hcl
# Create a DDS database user in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "database_user_name" {
  description = "The database user name"
  type        = string
}

resource "huaweicloud_dds_database_user" "test" {
  instance_id = huaweicloud_dds_instance.test.id
  name        = var.database_user_name
  password    = var.instance_password == "" ? try(random_password.test[0].result, null) : var.instance_password
  db_name     = "admin"

  roles {
    name    = huaweicloud_dds_database_role.test.name
    db_name = "admin"
  }
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the id of the resource huaweicloud_dds_instance.test
- **name**: Assigned by referencing the input variable database_user_name
- **password**: When the input variable instance_password is empty, it references the password generated by the random password resource, otherwise it is assigned by referencing the input variable instance_password
- **db_name**: The name of the database to which the database user belongs, fixed to admin
- **roles.name**: Assigned by referencing the name of the resource huaweicloud_dds_database_role.test
- **roles.db_name**: The name of the database to which the database role belongs, fixed to admin

### 10. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name            = "tf_test_database"
subnet_name         = "tf_test_database"
security_group_name = "tf_test_database"
instance_name       = "tf_test_database"
database_role_name  = "tf_test_database"
database_user_name  = "tf_test_database"
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

### 11. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DDS database role and user
4. Run `terraform show` to view the created DDS database role and user

## Reference Information

- [Huawei Cloud Document Database Service Product Documentation](https://support.huaweicloud.com/dds/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DDS Database Role and User](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/dds-database)
