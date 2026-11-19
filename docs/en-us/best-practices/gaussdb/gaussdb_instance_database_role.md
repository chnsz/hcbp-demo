# Deploy Instance Database Role

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud, supporting both centralized and distributed deployment modes. In real-world scenarios, database administrators need to create dedicated database roles for business systems so that database access can be isolated and controlled based on the principle of least privilege.

This best practice will introduce how to use Terraform to automatically deploy a GaussDB instance database role, including VPC creation, subnet configuration, security group configuration, GaussDB instance creation, and database role creation.

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
- [GaussDB Instance Database Role (huaweicloud_gaussdb_instance_database_role)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance_database_role)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_gaussdb_instance

data.huaweicloud_gaussdb_flavors
    └── huaweicloud_gaussdb_instance

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    │   └── huaweicloud_gaussdb_instance
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_networking_secgroup
        └── huaweicloud_gaussdb_instance

random_password
    ├── huaweicloud_gaussdb_instance
    └── huaweicloud_gaussdb_instance_database_role

huaweicloud_gaussdb_instance
    └── huaweicloud_gaussdb_instance_database_role
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration details.

### 2. Create a Virtual Private Cloud

Add the following script to the TF file (such as main.tf) to create a virtual private cloud:

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
- **cidr**: Assigned by referencing the input variable vpc_cidr, defaulting to `192.168.0.0/16`
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; the default enterprise project is used when it is empty

### 3. Create a Subnet

Add the following script to the TF file (such as main.tf) to create a subnet:

```hcl
# Create a subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **cidr**: Assigned by referencing the input variable subnet_cidr; automatically calculated from the VPC CIDR block when it is empty
- **gateway_ip**: Assigned by referencing the input variable gateway_ip; automatically calculated from the subnet CIDR block when it is empty

### 4. Query Availability Zones and GaussDB Flavors

Add the following script to the TF file (such as main.tf) to query the availability zones and GaussDB flavors:

```hcl
# Query the availability zones and GaussDB flavors data sources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_gaussdb_flavors" "test" {
  version = "8.201"
  ha_mode = "centralization_standard"
}
```

**Parameter Description**:
- **version**: The GaussDB instance version
- **ha_mode**: The GaussDB instance HA mode, which is centralized standard here

### 5. Create a Security Group and Security Group Rule

Add the following script to the TF file (such as main.tf) to create a security group and security group rule:

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
- **delete_default_rules**: Whether to delete the default rules of the security group, set to `true` here
- **security_group_id**: Assigned by referencing the ID of the security group resource
- **direction**: The direction of the security group rule, which is ingress here
- **protocol**: The protocol of the security group rule, which is TCP here
- **ports**: Assigned by referencing the input variable security_group_rule_ports to open the ports required by the GaussDB instance
- **remote_ip_prefix**: Assigned by referencing the CIDR block of the virtual private cloud resource

### 6. Create a Random Password

Add the following script to the TF file (such as main.tf) to automatically generate a random password when no password is specified:

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
- **length**: The length of the random password, which is 12 here
- **special**: Whether to include special characters, set to `true` here
- **override_special**: The set of allowed special characters
- **min_lower**, **min_upper**, **min_numeric**, **min_special**: The minimum number of characters of each type

### 7. Create a GaussDB Instance

Add the following script to the TF file (such as main.tf) to create a GaussDB instance:

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
- **flavor**: Assigned by referencing the spec code in the GaussDB flavors data source
- **name**: Assigned by referencing the input variable instance_name
- **password**: Assigned by referencing the input variable instance_password; the random password is used when it is empty
- **replica_num**: The number of instance replicas, which is 3 here
- **availability_zone**: Assigned by referencing the first three availability zones in the availability zones data source
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; the default enterprise project is used when it is empty
- **ha.mode**: The HA mode, which is centralized standard here
- **ha.replication_mode**: The replication mode, which is synchronous replication here
- **ha.consistency**: The consistency level, which is strong consistency here
- **ha.instance_mode**: The instance mode, which is basic here
- **volume.type**: Assigned by referencing the input variable instance_volume_type
- **volume.size**: Assigned by referencing the input variable instance_volume_size

### 8. Create a GaussDB Instance Database Role

Add the following script to the TF file (such as main.tf) to create a GaussDB instance database role:

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
- **instance_id**: Assigned by referencing the ID of the GaussDB instance resource
- **name**: Assigned by referencing the input variable database_role_name; the role name must be 1 to 63 characters, can contain letters, digits, and underscores, and cannot start with `pg` or a digit
- **password**: Assigned by referencing the input variable database_role_password; the random password is used when it is empty. The password must contain 8 to 32 characters and at least three types of the following characters: uppercase letters, lowercase letters, digits, and special characters (`~!@#%^*-_=+?`)

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB instance database role
4. Run `terraform show` to view the created GaussDB instance database role

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Instance Database Role](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-instance-database-role)
