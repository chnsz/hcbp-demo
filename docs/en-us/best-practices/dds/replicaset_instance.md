# Deploy Replica Set Instance

## Application Scenario

Document Database Service (DDS) is a high-performance, highly reliable, and secure distributed document database service provided by Huawei Cloud, fully compatible with the MongoDB protocol. The replica set architecture achieves automatic failover through multi-node data redundancy, making it a common deployment mode that balances cost and availability. It is suitable for business scenarios such as the Internet, IoT, gaming, and finance that have high requirements for data reliability.

This best practice will introduce how to use Terraform to automatically deploy a DDS replica set instance, including availability zone query, VPC creation, subnet configuration, security group configuration, and replica set instance creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DDS Flavors (data.huaweicloud_dds_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dds_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DDS Instance (huaweicloud_dds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dds_instance

data.huaweicloud_dds_flavors
    └── huaweicloud_dds_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_dds_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dds_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the article [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones available for the DDS instance:

```hcl
# Query availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

- **count**: The data source is created when the input variable availability_zone is empty, used to automatically obtain availability zones; if an availability zone is specified, it is not queried

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

- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr

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

- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The subnet CIDR block; when the input variable subnet_cidr is empty, it is automatically divided based on the VPC CIDR block
- **gateway_ip**: The subnet gateway IP; when the input variable subnet_gateway_ip is empty, it is automatically calculated based on the subnet CIDR block

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

- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group; set to true to customize access rules as needed

### 6. Query DDS Flavors

Add the following script in the TF file (such as main.tf) to query the flavors available for the DDS instance:

```hcl
# Query DDS flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "node_spec_code" {
  description = "The node specification code of the DDS instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "engine_name" {
  description = "The DB engine name of the DDS instance"
  type        = string
  default     = "DDS-Community"
}

variable "flavor_vcpus" {
  description = "The VCPUs of the flavor"
  type        = number
  default     = 2
}

variable "flavor_memory" {
  description = "The memory of the flavor"
  type        = number
  default     = 4
}

variable "node_type" {
  description = "The type of the DDS instance node"
  type        = string
  default     = "replica"
}

data "huaweicloud_dds_flavors" "test" {
  count = var.node_spec_code == "" ? 1 : 0

  engine_name = var.engine_name
  vcpus       = var.flavor_vcpus
  memory      = var.flavor_memory
  type        = var.node_type
}
```

**Parameter Description**:

- **count**: The data source is created when the input variable node_spec_code is empty, used to automatically query flavors; if a specification code is specified, it is not queried
- **engine_name**: The database engine name, assigned by referencing the input variable engine_name
- **vcpus**: The number of vCPUs of the flavor, assigned by referencing the input variable flavor_vcpus
- **memory**: The memory size of the flavor, assigned by referencing the input variable flavor_memory
- **type**: The node type, assigned by referencing the input variable node_type

### 7. Create a DDS Replica Set Instance

Add the following script in the TF file (such as main.tf) to create a DDS replica set instance:

```hcl
# Create a DDS replica set instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the DDS instance"
  type        = string
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

variable "node_number" {
  description = "The number of nodes of the DDS instance"
  type        = number
  default     = 3
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

variable "instance_port" {
  description = "The database access port of the DDS instance"
  type        = number
  default     = 8635
}

variable "instance_description" {
  description = "The description of the DDS instance"
  type        = string
  default     = ""
}

variable "instance_password" {
  description = "The database access password of the DDS instance"
  sensitive   = true
  type        = string
  default     = ""
}

variable "instance_tags" {
  description = "The tags of the DDS instance"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the DDS instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the DDS instance"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the DDS instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the DDS instance"
  type        = string
  default     = "false"
}

resource "huaweicloud_dds_instance" "test" {
  name              = var.instance_name
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  mode              = "ReplicaSet"

  datastore {
    type           = var.engine_name
    version        = var.database_version
    storage_engine = var.storage_engine
  }

  flavor {
    type      = var.node_type
    num       = var.node_number
    spec_code = var.node_spec_code == "" ? try(data.huaweicloud_dds_flavors.test[0].flavors[1].spec_code, null) : var.node_spec_code
    storage   = var.node_storage_type
    size      = var.node_size
    node_list = var.node_list
  }

  port          = var.instance_port
  description   = var.instance_description
  password      = var.instance_password
  tags          = var.instance_tags
  charging_mode = var.charging_mode
  period_unit   = var.period_unit
  period        = var.period
  auto_renew    = var.auto_renew
}
```

**Parameter Description**:

- **name**: The DDS instance name, assigned by referencing the input variable instance_name
- **availability_zone**: The availability zone to which the instance belongs; when the input variable availability_zone is empty, the first queried availability zone is used
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC created in the previous step
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet created in the previous step
- **security_group_id**: The ID of the security group to which the instance belongs, referencing the ID of the security group created in the previous step
- **mode**: The instance deployment mode, set to ReplicaSet to indicate a replica set
- **datastore.type**: The database engine type, assigned by referencing the input variable engine_name
- **datastore.version**: The database version, assigned by referencing the input variable database_version
- **datastore.storage_engine**: The storage engine, assigned by referencing the input variable storage_engine
- **flavor.type**: The node type, assigned by referencing the input variable node_type
- **flavor.num**: The number of nodes, assigned by referencing the input variable node_number
- **flavor.spec_code**: The specification code; when the input variable node_spec_code is empty, the queried specification code is used
- **flavor.storage**: The node storage type, assigned by referencing the input variable node_storage_type
- **flavor.size**: The node disk size, assigned by referencing the input variable node_size
- **flavor.node_list**: The list of node IDs to be deleted, assigned by referencing the input variable node_list
- **port**: The database access port, assigned by referencing the input variable instance_port
- **description**: The instance description, assigned by referencing the input variable instance_description
- **password**: The database access password, assigned by referencing the input variable instance_password
- **tags**: The instance tags, assigned by referencing the input variable instance_tags
- **charging_mode**: The charging mode, assigned by referencing the input variable charging_mode
- **period_unit**: The charging period unit, assigned by referencing the input variable period_unit
- **period**: The charging period, assigned by referencing the input variable period
- **auto_renew**: Whether to enable auto renewal, assigned by referencing the input variable auto_renew

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
vpc_name            = "tf_test_instance"
subnet_name         = "tf_test_instance"
security_group_name = "tf_test_instance"
instance_name       = "tf_test_instance"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DDS replica set instance
4. Run `terraform show` to view the created DDS replica set instance

## Reference Information

- [Huawei Cloud Document Database Service Product Documentation](https://support.huaweicloud.com/dds/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DDS Replica Set Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/dds-instance/replicaset-instance)
