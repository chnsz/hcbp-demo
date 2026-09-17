# Deploy Readonly Node

## Application Scenario

Document Database Service (DDS) replica set instances support adding readonly nodes to an instance, which can be used to offload read requests from the primary node and improve the read concurrency of the database. Readonly nodes keep data synchronized with the primary node and are suitable for read-heavy business scenarios, such as report queries, data analysis, and historical data retrieval.

This best practice will introduce how to use Terraform to automatically create a readonly node for a DDS replica set instance, including availability zone query, VPC creation, subnet configuration, security group configuration, DDS replica set instance creation, and readonly node creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Network ACL (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DDS Instance (huaweicloud_dds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)
- [DDS Readonly Node (huaweicloud_dds_readonly_node)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_readonly_node)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dds_instance
        └── huaweicloud_dds_readonly_node

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dds_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dds_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [preparation before deploying Huawei Cloud resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf) to query availability zones:

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

**Parameter description**:
- **count**: Assigned by referencing the input variable availability_zone, queries the availability zone list when no availability zone is specified

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

**Parameter description**:
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr

### 4. Create a VPC Subnet

Add the following script in the TF file (such as main.tf) to create a VPC subnet:

```hcl
# Create a VPC subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter description**:
- **vpc_id**: Assigned by referencing the ID of the resource huaweicloud_vpc.test
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr, the subnet CIDR is automatically divided based on the VPC CIDR when no subnet CIDR is specified
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip, the gateway IP is automatically calculated based on the subnet CIDR when no gateway IP is specified

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

**Parameter description**:
- **name**: Assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group, set to true to delete the default rules

### 6. Create a DDS Replica Set Instance

Add the following script in the TF file (such as main.tf) to create a DDS replica set instance:

```hcl
# Create a DDS replica set instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the DDS instance"
  type        = string
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

resource "huaweicloud_dds_instance" "test" {
  name              = var.instance_name
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  mode              = "ReplicaSet"

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
  }
}
```

**Parameter description**:
- **name**: Assigned by referencing the input variable instance_name
- **availability_zone**: Assigned by referencing the input variable availability_zone, the first availability zone in the queried availability zone list is used when no availability zone is specified
- **vpc_id**: Assigned by referencing the ID of the resource huaweicloud_vpc.test
- **subnet_id**: Assigned by referencing the ID of the resource huaweicloud_vpc_subnet.test
- **security_group_id**: Assigned by referencing the ID of the resource huaweicloud_networking_secgroup.test
- **mode**: The instance mode, set to ReplicaSet to create a replica set instance
- **datastore.type**: Assigned by referencing the input variable database_type
- **datastore.version**: Assigned by referencing the input variable database_version
- **datastore.storage_engine**: Assigned by referencing the input variable storage_engine
- **flavor.type**: Assigned by referencing the input variable node_type
- **flavor.num**: Assigned by referencing the input variable node_number
- **flavor.spec_code**: Assigned by referencing the input variable node_spec_code
- **flavor.storage**: Assigned by referencing the input variable node_storage_type
- **flavor.size**: Assigned by referencing the input variable node_size

### 7. Create a DDS Readonly Node

Add the following script in the TF file (such as main.tf) to create a DDS readonly node:

```hcl
# Create a DDS readonly node in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "node_spec" {
  description = "The spec code of the read only node"
  type        = string
  default     = "dds.mongodb.s6.large.4.rr"
  nullable    = false
}

resource "huaweicloud_dds_readonly_node" "test" {
  instance_id = huaweicloud_dds_instance.test.id
  spec_code   = var.node_spec
}
```

**Parameter description**:
- **instance_id**: Assigned by referencing the ID of the resource huaweicloud_dds_instance.test
- **spec_code**: Assigned by referencing the input variable node_spec

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
vpc_name            = "tf_test_node"
subnet_name         = "tf_test_node"
security_group_name = "tf_test_node"
instance_name       = "tf_test_node"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DDS readonly node
4. Run `terraform show` to view the created DDS readonly node

## Reference Information

- [Huawei Cloud Document Database Service Product Documentation](https://support.huaweicloud.com/dds/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DDS Readonly Node](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/readonly-node)
