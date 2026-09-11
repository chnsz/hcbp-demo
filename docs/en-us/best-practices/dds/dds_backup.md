# Deploy DDS Instance Backup

## Application Scenario

Document Database Service (DDS) is a high-performance, highly reliable, and secure distributed document database service provided by Huawei Cloud, fully compatible with the MongoDB protocol. In real-world business scenarios, to prevent data loss caused by accidental deletion, data corruption, or logical errors, DDS instances need to be backed up regularly so that data can be quickly restored when exceptions occur, ensuring business continuity.

This best practice will introduce how to use Terraform to automatically deploy a DDS instance backup, including availability zone query, VPC creation, subnet configuration, security group configuration, DDS instance creation, and backup creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DDS Instance (huaweicloud_dds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)
- [DDS Backup (huaweicloud_dds_backup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_backup)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dds_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet

huaweicloud_vpc
huaweicloud_vpc_subnet
huaweicloud_networking_secgroup
    └── huaweicloud_dds_instance
        └── huaweicloud_dds_backup
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zone Information

Add the following script in the TF file (such as main.tf) to query the availability zones available for the DDS instance:

```hcl
# Query availability zone information in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone to which the DDS backup belongs"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: Assigned by referencing the input variable availability_zone, queries the availability zone list when no availability zone is specified

### 3. Create a Virtual Private Cloud

Add the following script in the TF file to create a VPC:

```hcl
# Create a virtual private cloud in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

### 4. Create a Virtual Private Cloud Subnet

Add the following script in the TF file to create a subnet:

```hcl
# Create a virtual private cloud subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, associates the VPC to which the subnet belongs
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr, automatically divides the subnet based on the VPC CIDR when not specified
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip, automatically calculates the gateway IP when not specified

### 5. Create a Security Group

Add the following script in the TF file to create a security group:

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

### 6. Create a DDS Instance

Add the following script in the TF file to create a DDS instance:

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
- **availability_zone**: Assigned by referencing the input variable availability_zone, uses the first queried availability zone when not specified
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id
- **mode**: Assigned by referencing the input variable instance_mode
- **datastore.type**: Assigned by referencing the input variable database_type
- **datastore.version**: Assigned by referencing the input variable database_version
- **datastore.storage_engine**: Assigned by referencing the input variable storage_engine
- **flavor.type**: Assigned by referencing the input variable node_type
- **flavor.num**: Assigned by referencing the input variable node_number
- **flavor.spec_code**: Assigned by referencing the input variable node_spec_code
- **flavor.storage**: Assigned by referencing the input variable node_storage_type
- **flavor.size**: Assigned by referencing the input variable node_size
- **flavor.node_list**: Assigned by referencing the input variable node_list

### 7. Create a DDS Backup

Add the following script in the TF file to create a DDS backup:

```hcl
# Create a DDS backup in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "backup_name" {
  description = "The name of the DDS backup"
  type        = string
}

variable "backup_description" {
  description = "The description of the DDS backup"
  type        = string
  default     = ""
}

resource "huaweicloud_dds_backup" "test" {
  instance_id = huaweicloud_dds_instance.test.id
  name        = var.backup_name
  description = var.backup_description
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing huaweicloud_dds_instance.test.id, associates the DDS instance to be backed up
- **name**: Assigned by referencing the input variable backup_name
- **description**: Assigned by referencing the input variable backup_description

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "your_region_name"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name            = "tf_test_backup"
subnet_name         = "tf_test_backup"
security_group_name = "tf_test_backup"
instance_name       = "tf_test_backup"
backup_name         = "tf_test_backup"
backup_description  = "This is a backup created by terraform"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DDS instance backup
4. Run `terraform show` to view the created DDS instance backup

## Reference Information

- [Huawei Cloud Document Database Service Product Documentation](https://support.huaweicloud.com/dds/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DDS Instance Backup](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/dds-backup)
