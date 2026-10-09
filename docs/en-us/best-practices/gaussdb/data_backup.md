# Deploy Data Backup

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud, supporting both centralized and distributed deployment modes. In real-world business scenarios, to prevent data loss caused by misoperations, software failures, or data corruption, database instances need to be backed up regularly so that data can be restored to a specified backup point when needed.

This best practice will introduce how to use Terraform to automatically deploy a GaussDB instance and create a manual backup for it, including the creation of network resources such as VPC, subnet, and security group, GaussDB instance configuration, and manual backup creation.

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
- [GaussDB Backup (huaweicloud_gaussdb_backup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_backup)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_gaussdb_instance.test

huaweicloud_vpc.test
    ├── huaweicloud_vpc_subnet.test
    │   └── huaweicloud_gaussdb_instance.test
    └── huaweicloud_networking_secgroup_rule.test

huaweicloud_networking_secgroup.test
    ├── huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_gaussdb_instance.test

random_password.test
    └── huaweicloud_gaussdb_instance.test

huaweicloud_gaussdb_instance.test
    └── huaweicloud_gaussdb_backup.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Virtual Private Cloud

Add the following script in the TF file (such as main.tf) to create a virtual private cloud:

```hcl
# Create a virtual private cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The VPC name"
  type        = string
  nullable    = false
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  nullable    = false
  default     = "172.16.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr

### 3. Create a Virtual Private Cloud Subnet

Add the following script in the TF file (such as main.tf) to create a virtual private cloud subnet:

```hcl
# Create a virtual private cloud subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The subnet name"
  type        = string
  nullable    = false
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  nullable    = false
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  nullable    = false
  default     = ""
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing the ID of the virtual private cloud resource
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; if empty, it is calculated from the VPC CIDR
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip; if empty, it is calculated from the subnet CIDR

### 4. Create a Security Group

Add the following script in the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name"
  type        = string
  nullable    = false
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable security_group_name
- **delete_default_rules**: Set to true to delete the default rules of the security group so that rules can be added as needed

### 5. Create a Security Group Rule

Add the following script in the TF file (such as main.tf) to create a security group rule:

```hcl
# Create a security group rule resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_rule_ports" {
  description = "The security group ingress rule ports"
  type        = string
  nullable    = false
  default     = "2379-2380,5000-5001,5432-5532,6000,6500,12016,20050"
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = huaweicloud_vpc.test.cidr
  ports             = var.security_group_rule_ports
  protocol          = "tcp"
}
```

**Parameter Description**:
- **security_group_id**: Assigned by referencing the ID of the security group resource
- **direction**: Set to ingress to indicate an inbound rule
- **ethertype**: Set to IPv4 to indicate the IPv4 protocol
- **remote_ip_prefix**: Assigned by referencing the CIDR of the virtual private cloud
- **ports**: Assigned by referencing the input variable security_group_rule_ports, used to open the ports required by the GaussDB instance
- **protocol**: Set to tcp to indicate the TCP protocol

### 6. Query the Availability Zone List

Add the following script in the TF file (such as main.tf) to query the availability zone list:

```hcl
# Query the availability zone list data source in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_availability_zones" {
  description = "The availability zones for the GaussDB instance, separated by commas"
  type        = string
  nullable    = false
  default     = ""
}

data "huaweicloud_availability_zones" "test" {
  count = var.instance_availability_zones == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable instance_availability_zones is empty, used to automatically obtain the availability zone list

### 7. Create a Random Password

Add the following script in the TF file (such as main.tf) to create a random password:

```hcl
# Create a random password resource
resource "random_password" "test" {
  length           = 16
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
  special          = true
  override_special = "~!@#%^*-_=+?"
}
```

**Parameter Description**:
- **length**: The password length, set to 16
- **min_upper**: The minimum number of uppercase letters, set to 1
- **min_lower**: The minimum number of lowercase letters, set to 1
- **min_numeric**: The minimum number of digits, set to 1
- **min_special**: The minimum number of special characters, set to 1
- **special**: Set to true to allow special characters
- **override_special**: The set of allowed special characters

### 8. Create a GaussDB Instance

Add the following script in the TF file (such as main.tf) to create a GaussDB instance:

```hcl
# Create a GaussDB instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the GaussDB instance"
  type        = string
  nullable    = false
}

variable "instance_flavor" {
  description = "The flavor of the GaussDB instance"
  type        = string
  nullable    = false
  default     = "gaussdb.opengauss.ee.c3.xlarge.x864.ha"
}

variable "instance_password" {
  description = "The password for the GaussDB instance"
  type        = string
  sensitive   = true
  nullable    = false
  default     = ""
}

variable "instance_db_port" {
  description = "The database port of the GaussDB instance"
  type        = number
  nullable    = false
  default     = 5432
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the GaussDB instance"
  type        = string
  default     = null
}

variable "instance_ha_mode" {
  description = "The HA mode of the GaussDB instance"
  type        = string
  nullable    = false
  default     = "centralization_standard"
}

variable "instance_ha_replication_mode" {
  description = "The HA replication mode of the GaussDB instance"
  type        = string
  nullable    = false
  default     = "sync"
}

variable "instance_ha_consistency" {
  description = "The HA consistency of the GaussDB instance"
  type        = string
  nullable    = false
  default     = "strong"
}

variable "instance_volume_type" {
  description = "The storage volume type of the GaussDB instance"
  type        = string
  nullable    = false
  default     = "ULTRAHIGH"
}

variable "instance_volume_size" {
  description = "The storage volume size (GB) of the GaussDB instance"
  type        = number
  nullable    = false
  default     = 40
}

resource "huaweicloud_gaussdb_instance" "test" {
  name                  = var.instance_name
  flavor                = var.instance_flavor
  password              = var.instance_password != "" ? var.instance_password : random_password.test.result
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zone     = var.instance_availability_zones != "" ? var.instance_availability_zones : join(",", slice(data.huaweicloud_availability_zones.test[0].names, 0, 3))
  port                  = var.instance_db_port
  enterprise_project_id = var.enterprise_project_id

  ha {
    mode             = var.instance_ha_mode
    replication_mode = var.instance_ha_replication_mode
    consistency      = var.instance_ha_consistency
  }

  replica_num = 3

  volume {
    type = var.instance_volume_type
    size = var.instance_volume_size
  }

  lifecycle {
    ignore_changes = [
      flavor,
    ]
  }
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable instance_name
- **flavor**: Assigned by referencing the input variable instance_flavor
- **password**: Assigned by referencing the input variable instance_password; if empty, the result generated by the random password resource is used
- **vpc_id**: Assigned by referencing the ID of the virtual private cloud resource
- **subnet_id**: Assigned by referencing the ID of the virtual private cloud subnet resource
- **security_group_id**: Assigned by referencing the ID of the security group resource
- **availability_zone**: Assigned by referencing the input variable instance_availability_zones; if empty, the first 3 availability zones are used automatically
- **port**: Assigned by referencing the input variable instance_db_port
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id
- **ha.mode**: Assigned by referencing the input variable instance_ha_mode
- **ha.replication_mode**: Assigned by referencing the input variable instance_ha_replication_mode
- **ha.consistency**: Assigned by referencing the input variable instance_ha_consistency
- **replica_num**: The number of replicas, set to 3
- **volume.type**: Assigned by referencing the input variable instance_volume_type
- **volume.size**: Assigned by referencing the input variable instance_volume_size
- **lifecycle.ignore_changes**: Ignores changes to the flavor field to prevent unintended instance specification modifications

### 9. Create a GaussDB Manual Backup

Add the following script in the TF file (such as main.tf) to create a GaussDB manual backup:

```hcl
# Create a GaussDB backup resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "backup_name" {
  description = "The name for the manual backup"
  type        = string
  nullable    = false
}

variable "backup_description" {
  description = "The description for the manual backup"
  type        = string
  nullable    = false
  default     = ""
}

resource "huaweicloud_gaussdb_backup" "test" {
  instance_id = huaweicloud_gaussdb_instance.test.id
  name        = var.backup_name
  description = var.backup_description
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the ID of the GaussDB instance resource
- **name**: Assigned by referencing the input variable backup_name
- **description**: Assigned by referencing the input variable backup_description

### 10. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Network resource variables
vpc_name            = "your_vpc_name"
subnet_name         = "your_subnet_name"
security_group_name = "your_security_group_name"

# GaussDB instance variables
instance_name = "your_gaussdb_instance_name"

# Manual backup variables
backup_name = "your_manual_backup_name"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB instance and its manual backup
4. Run `terraform show` to view the created GaussDB instance and its manual backup

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Data Backup](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/data-backup)
