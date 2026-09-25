# Deploy GeminiDB Cassandra Instance

## Application Scenario

GeminiDB Cassandra is a distributed NoSQL database service provided by Huawei Cloud that is compatible with the Apache Cassandra protocol. It offers high availability, high reliability, and elastic scaling, making it suitable for scenarios such as massive data storage, high-concurrency read and write, and wide-column data models.

This best practice will introduce how to use Terraform to automatically deploy a GeminiDB Cassandra instance, including the creation of a VPC, subnet, and security group, automatic querying of instance flavors, automatic generation of the instance password, and configuration of the backup strategy and instance backup.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [GeminiDB NoSQL Flavors (data.huaweicloud_gaussdb_nosql_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_nosql_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GeminiDB Instance (huaweicloud_geminidb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/geminidb_instance)
- [GeminiDB Backup (huaweicloud_geminidb_backup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/geminidb_backup)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    ├── data.huaweicloud_gaussdb_nosql_flavors.test
    └── huaweicloud_geminidb_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
            └── huaweicloud_geminidb_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_geminidb_instance.test

random_password.test
    └── huaweicloud_geminidb_instance.test

huaweicloud_geminidb_instance.test
    └── huaweicloud_geminidb_backup.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

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
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC resource
- **cidr**: The CIDR block of the subnet. When the input variable subnet_cidr is empty, it is automatically calculated based on the VPC CIDR using the cidrsubnet function
- **gateway_ip**: The gateway IP address of the subnet. When the input variable gateway_ip is empty, it is automatically calculated based on the subnet CIDR using the cidrhost function

### 3. Query Availability Zones and GeminiDB NoSQL Flavors

Add the following script in the TF file (such as main.tf) to query availability zones and GeminiDB NoSQL flavors:

```hcl
# Query availability zones and GeminiDB NoSQL flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vcpus" {
  description = "The number of vCPUs"
  type        = string
  default     = "2"
}

variable "availability_zone" {
  description = "The availability zone to which the GeminiDB Cassandra instance belongs"
  type        = string
  default     = ""
}

data "huaweicloud_availability_zones" "test" {
}

data "huaweicloud_gaussdb_nosql_flavors" "test" {
  vcpus             = var.vcpus
  engine            = "cassandra"
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **vcpus**: The number of vCPUs of the flavor, assigned by referencing the input variable vcpus
- **engine**: The database engine type, fixed to cassandra
- **availability_zone**: The availability zone to which the flavor belongs. When the input variable availability_zone is empty, the first availability zone in the availability zone list is used automatically

### 4. Create Security Group

Add the following script in the TF file (such as main.tf) to create a security group:

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
- **delete_default_rules**: Whether to delete the default rules of the security group. It is set to true to allow custom access rules as needed

### 5. Generate Random Password

Add the following script in the TF file (such as main.tf) to automatically generate a random password when no password is provided:

```hcl
# Generate a random password in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_password" {
  description = "The password for the GeminiDB Cassandra instance"
  type        = string
  default     = ""
  sensitive   = true
}

resource "random_password" "test" {
  count            = var.instance_password == "" ? 1 : 0

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
- **count**: Creates a random password when the input variable instance_password is empty, otherwise it is not created
- **length**: The password length, set to 12
- **special**: Whether to include special characters, set to true
- **override_special**: The set of special characters allowed
- **min_upper**, **min_lower**, **min_numeric**, **min_special**: Specify the minimum number of uppercase letters, lowercase letters, digits, and special characters respectively

### 6. Create GeminiDB Cassandra Instance

Add the following script in the TF file (such as main.tf) to create a GeminiDB Cassandra instance:

```hcl
# Create a GeminiDB Cassandra instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The GeminiDB Cassandra instance name"
  type        = string
}

variable "instance_mode" {
  description = "The instance mode. Valid values are Cluster, Single"
  type        = string
  default     = "Cluster"
}

variable "instance_db_port" {
  description = "The Cassandra database port"
  type        = number
  default     = 9042
}

variable "instance_ssl_option" {
  description = "The SSL option. Valid values are on, off"
  type        = string
  default     = "on"
}

variable "instance_flavor_num" {
  description = "The number of nodes in the Cassandra cluster"
  type        = number
  default     = 3
}

variable "instance_flavor_size" {
  description = "The storage size in GB per node"
  type        = number
  default     = 100
}

variable "instance_flavor_storage" {
  description = "The storage type. Valid values are ULTRAHIGH, ESSD"
  type        = string
  default     = "ULTRAHIGH"
}

variable "instance_flavor_spec_code" {
  description = "The resource specification code. If empty, it will be queried from flavors data source"
  type        = string
  default     = ""
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
  description = "The key/value pairs to associate with the GeminiDB Cassandra instance"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_geminidb_instance" "test" {
  name              = var.instance_name
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  password          = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result)
  mode              = var.instance_mode
  port              = var.instance_db_port
  ssl_option        = var.instance_ssl_option

  datastore {
    type           = "cassandra"
    version        = "3.11"
    storage_engine = "rocksDB"
  }

  flavor {
    num       = var.instance_flavor_num
    size      = var.instance_flavor_size
    storage   = var.instance_flavor_storage
    spec_code = var.instance_flavor_spec_code != "" ? var.instance_flavor_spec_code : try(data.huaweicloud_gaussdb_nosql_flavors.test.flavors[0].name, null)
  }

  backup_strategy {
    start_time = var.instance_backup_time_window
    keep_days  = var.instance_backup_keep_days
  }

  charging_mode = "prePaid"
  period_unit   = "month"
  auto_renew    = "true"
  period        = 1

  tags = var.tags

  lifecycle {
    ignore_changes = [
      flavor.0.spec_code,
    ]
  }
}
```

**Parameter Description**:
- **name**: The instance name, assigned by referencing the input variable instance_name
- **availability_zone**: The availability zone to which the instance belongs. When the input variable availability_zone is empty, the first availability zone in the availability zone list is used automatically
- **vpc_id**: The ID of the VPC to which the instance belongs, assigned by referencing the ID of the VPC resource
- **subnet_id**: The ID of the subnet to which the instance belongs, assigned by referencing the ID of the subnet resource
- **security_group_id**: The ID of the security group to which the instance belongs, assigned by referencing the ID of the security group resource
- **password**: The instance password. When the input variable instance_password is not empty, this value is used; otherwise, the automatically generated random password is used
- **mode**: The instance mode, assigned by referencing the input variable instance_mode
- **port**: The database port, assigned by referencing the input variable instance_db_port
- **ssl_option**: The SSL option, assigned by referencing the input variable instance_ssl_option
- **datastore**: The database engine information, with type set to cassandra, version set to 3.11, and storage_engine set to rocksDB
- **flavor**: The instance flavor information. num, size, and storage are assigned by referencing the input variables instance_flavor_num, instance_flavor_size, and instance_flavor_storage respectively; spec_code is automatically obtained from the flavors data source when the input variable instance_flavor_spec_code is empty
- **backup_strategy**: The backup strategy. start_time and keep_days are assigned by referencing the input variables instance_backup_time_window and instance_backup_keep_days respectively
- **charging_mode**, **period_unit**, **auto_renew**, **period**: Parameters related to the charging mode, set to prePaid, month, true, and 1 respectively
- **tags**: The instance tags, assigned by referencing the input variable tags
- **lifecycle**: The lifecycle configuration, ignoring changes to spec_code in flavor

### 7. Create GeminiDB Backup

Add the following script in the TF file (such as main.tf) to create a GeminiDB backup:

```hcl
# Create a GeminiDB backup in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "backup_name" {
  description = "The name for instance backups"
  type        = string
}

variable "backup_description" {
  description = "The description for instance backups"
  type        = string
  default     = "Terraform created backup"
}

resource "huaweicloud_geminidb_backup" "test" {
  instance_id = huaweicloud_geminidb_instance.test.id
  name        = var.backup_name
  description = var.backup_description

  depends_on = [huaweicloud_geminidb_instance.test]
}
```

**Parameter Description**:
- **instance_id**: The ID of the instance to which the backup belongs, assigned by referencing the ID of the GeminiDB instance resource
- **name**: The backup name, assigned by referencing the input variable backup_name
- **description**: The backup description, assigned by referencing the input variable backup_description
- **depends_on**: Explicitly declares the dependency to ensure the backup is created after the instance is created

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name                    = "tf_test_vpc"
subnet_name                 = "tf_test_subnet"
security_group_name         = "tf_test_security_group"
instance_name               = "tf_test_geminidb_cassandra"
instance_backup_time_window = "03:00-04:00"
instance_backup_keep_days   = 14
backup_name                 = "tf_test_backup"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GeminiDB Cassandra instance
4. Run `terraform show` to view the created GeminiDB Cassandra instance

## Reference Information

- [Huawei Cloud GeminiDB Product Documentation](https://support.huaweicloud.com/geminidb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GeminiDB Cassandra Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/geminidb/geminidb-cassandra-instance)
