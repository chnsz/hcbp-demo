# Deploy Instance Database Account

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud. In real-world business scenarios, after a database instance is created, you usually need to create dedicated database accounts for business applications, and the `is_login_only` parameter can be used to control whether an account is used for login only, thereby achieving permission isolation and least-privilege management.

This best practice will introduce how to use Terraform to automatically deploy a GaussDB instance database account, including VPC creation, subnet configuration, security group configuration, GaussDB instance creation, and database account creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [GaussDB Flavors (data.huaweicloud_gaussdb_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB Instance (huaweicloud_gaussdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB Instance Database Account (huaweicloud_gaussdb_instance_database_account)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance_database_account)

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
    └── huaweicloud_gaussdb_instance

random_password
    └── huaweicloud_gaussdb_instance

huaweicloud_gaussdb_instance
    └── huaweicloud_gaussdb_instance_database_account
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Virtual Private Cloud

Add the following script to the TF file (such as main.tf) to create a Virtual Private Cloud:

```hcl
# Create a Virtual Private Cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr, defaulting to `192.168.0.0/16`
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, and not specified when empty

### 3. Create a Virtual Private Cloud Subnet

Add the following script to the TF file (such as main.tf) to create a subnet:

```hcl
# Create a Virtual Private Cloud Subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet, assigned by referencing the input variable subnet_cidr, and automatically calculated based on the VPC CIDR when empty
- **gateway_ip**: The gateway IP address of the subnet, assigned by referencing the input variable gateway_ip, and automatically calculated based on the subnet CIDR when empty

### 4. Query Availability Zones and GaussDB Flavors

Add the following script to the TF file (such as main.tf) to query the availability zone list and GaussDB flavor list:

```hcl
# Query the availability zone list and GaussDB flavor list in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_gaussdb_flavors" "test" {
  version = "8.201"
  ha_mode = "centralization_standard"
}
```

**Parameter Description**:
- **version**: The GaussDB version, set to `8.201` here
- **ha_mode**: The instance deployment mode, set to `centralization_standard` (centralized standard edition) here

### 5. Create a Security Group

Add the following script to the TF file (such as main.tf) to create a security group:

```hcl
# Create a Security Group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name

### 6. Create a Random Password

Add the following script to the TF file (such as main.tf) to automatically generate a random password when the instance password is not specified:

```hcl
# Create a Random Password resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**Parameter Description**:
- **count**: Creates a random password when the input variable instance_password is empty, otherwise not created
- **length**: The length of the random password, set to 12 here
- **special**: Whether to include special characters, set to `true` here
- **override_special**: The set of allowed special characters
- **min_upper**/**min_lower**/**min_numeric**/**min_special**: The minimum number of uppercase letters, lowercase letters, digits, and special characters

### 7. Create a GaussDB Instance

Add the following script to the TF file (such as main.tf) to create a GaussDB instance:

```hcl
# Create a GaussDB Instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  availability_zone = join(",", [
    try(data.huaweicloud_availability_zones.test.names[0], ""),
    try(data.huaweicloud_availability_zones.test.names[1], ""),
    try(data.huaweicloud_availability_zones.test.names[2], "")
  ])

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
- **vpc_id**: The ID of the VPC to which the instance belongs, assigned by referencing the ID of the VPC created in the previous step
- **subnet_id**: The ID of the subnet to which the instance belongs, assigned by referencing the ID of the subnet created in the previous step
- **security_group_id**: The ID of the security group to which the instance belongs, assigned by referencing the ID of the security group created in the previous step
- **flavor**: The instance flavor, assigned by referencing the `spec_code` of the first flavor in the GaussDB flavor list data source
- **name**: The instance name, assigned by referencing the input variable instance_name
- **password**: The instance password, assigned by referencing the input variable instance_password, and using the random password when empty
- **replica_num**: The number of replicas, set to 3 here
- **availability_zone**: The availability zone information, assigned by referencing the first three availability zones in the availability zone list data source
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, and not specified when empty
- **ha.mode**: The instance deployment mode, set to `centralization_standard` here
- **ha.replication_mode**: The data replication mode, set to `sync` (synchronous replication) here
- **ha.consistency**: The consistency level, set to `strong` (strong consistency) here
- **ha.instance_mode**: The instance mode, set to `basic` here
- **volume.type**: The storage type, assigned by referencing the input variable instance_volume_type
- **volume.size**: The storage size, assigned by referencing the input variable instance_volume_size

### 8. Create a GaussDB Instance Database Account

Add the following script to the TF file (such as main.tf) to create a GaussDB instance database account:

```hcl
# Create a GaussDB Instance Database Account resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "database_account_name" {
  description = "The name of the database account"
  type        = string
}

variable "database_account_password" {
  description = "The password of the database account"
  type        = string
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
  password      = var.database_account_password
  is_login_only = var.is_login_only
}
```

**Parameter Description**:
- **instance_id**: The ID of the GaussDB instance to which the database account belongs, assigned by referencing the ID of the GaussDB instance created in the previous step
- **name**: The database account name, assigned by referencing the input variable database_account_name. It must be 1 to 63 characters, can contain letters, digits, and underscores, cannot start with `pg` or a digit, and cannot be the same as system user names (`rdsAdmin`, `rdsMetric`, `rdsBackup`, `rdsRepl`, `root`)
- **password**: The database account password, assigned by referencing the input variable database_account_password. It must contain 8 to 32 characters and at least three types of uppercase letters, lowercase letters, digits, and special characters (`~!@#%^*-_=+?`), and cannot be the same as the account name or the reverse of the account name
- **is_login_only**: Whether the database account supports login only, assigned by referencing the input variable is_login_only, with valid values `true` and `false`

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
vpc_name                  = "tf_test_vpc"
subnet_name               = "tf_test_subnet"
security_group_name       = "tf_test_security_group"
instance_name             = "tf_test_gaussdb_instance"
database_account_name     = "tf_test_account_name"
database_account_password = "your_account_password"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB instance database account
4. Run `terraform show` to view the created GaussDB instance database account

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Instance Database Account](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-instance-database-account)
