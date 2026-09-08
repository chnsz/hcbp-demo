# Deploy DDS Instance Associate EIP

## Application Scenario

Document Database Service (DDS) is a high-performance, highly reliable, and secure distributed document database service provided by Huawei Cloud. It is fully compatible with the MongoDB protocol and is suitable for various business scenarios. In real business, users may need to access DDS instances from the public network, such as for data migration, remote O&M, or business system integration.

This best practice will introduce how to use Terraform to create a DDS instance and associate it with an Elastic IP (EIP), enabling secure public network access to the DDS instance. Through this practice, you can learn how to use Terraform to automate the deployment of VPC, subnet, security group, EIP, and DDS instance resources, and complete the association between the DDS instance and the EIP.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DDS Instances (data.huaweicloud_dds_instances)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dds_instances)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [DDS Instance (huaweicloud_dds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)
- [DDS Instance EIP Associate (huaweicloud_dds_instance_eip_associate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance_eip_associate)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dds_instance
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
huaweicloud_vpc_subnet
    └── huaweicloud_dds_instance
huaweicloud_networking_secgroup
    └── huaweicloud_dds_instance
huaweicloud_vpc_eip
    └── huaweicloud_dds_instance_eip_associate
huaweicloud_dds_instance
    ├── data.huaweicloud_dds_instances
    └── huaweicloud_dds_instance_eip_associate
data.huaweicloud_dds_instances
    └── local.nodeId
local.nodeId
    └── huaweicloud_dds_instance_eip_associate
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the best practice script in the specified working directory, and ensure that it (or other TF files in the same level directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script to the TF file (such as main.tf) to query the available availability zones when no availability zone is specified:

```hcl
# Query availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: When the input variable availability_zone is an empty string, create this data source to query availability zones; otherwise, do not create it.

### 3. Create VPC

Add the following script to the TF file (such as main.tf) to create a VPC:

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
- **name**: Assigned by referencing the input variable vpc_name, specifying the VPC name.
- **cidr**: Assigned by referencing the input variable vpc_cidr, specifying the CIDR block of the VPC.

### 4. Create Subnet

Add the following script to the TF file (such as main.tf) to create a subnet:

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
- **vpc_id**: Assigned by referencing the id of the created VPC resource huaweicloud_vpc.test, specifying the VPC to which the subnet belongs.
- **name**: Assigned by referencing the input variable subnet_name, specifying the subnet name.
- **cidr**: Assigned by referencing the input variable subnet_cidr. When not specified, a subnet segment is automatically divided from the VPC CIDR.
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip. When not specified, the gateway address of the subnet is automatically calculated.

### 5. Create Security Group

Add the following script to the TF file (such as main.tf) to create a security group:

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
- **name**: Assigned by referencing the input variable security_group_name, specifying the security group name.
- **delete_default_rules**: Set to true to delete the default rules of the security group, facilitating subsequent custom rules.

### 6. Create EIP

Add the following script to the TF file (such as main.tf) to create an EIP:

```hcl
# Create an EIP in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "eip_bandwidth_name" {
  description = "The name of the EIP bandwidth"
  type        = string
}

variable "eip_bandwidth_size" {
  description = "The size of the EIP bandwidth in Mbit/s"
  type        = number
  default     = 5
}

variable "eip_bandwidth_charge_mode" {
  description = "The charge mode of the EIP bandwidth"
  type        = string
  default     = "traffic"
}

resource "huaweicloud_vpc_eip" "test" {
  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.eip_bandwidth_name
    share_type  = "PER"
    size        = var.eip_bandwidth_size
    charge_mode = var.eip_bandwidth_charge_mode
  }
}
```

**Parameter Description**:
- **publicip.type**: Assigned by referencing the input variable eip_type, specifying the type of the EIP.
- **bandwidth.name**: Assigned by referencing the input variable eip_bandwidth_name, specifying the bandwidth name.
- **bandwidth.share_type**: Fixed to "PER", indicating exclusive bandwidth.
- **bandwidth.size**: Assigned by referencing the input variable eip_bandwidth_size, specifying the bandwidth size.
- **bandwidth.charge_mode**: Assigned by referencing the input variable eip_bandwidth_charge_mode, specifying the bandwidth charge mode.

### 7. Create DDS Instance

Add the following script to the TF file (such as main.tf) to create a DDS instance:

```hcl
# Create a DDS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone to which the DDS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

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
- **name**: Assigned by referencing the input variable instance_name, specifying the DDS instance name.
- **availability_zone**: Assigned by referencing the input variable availability_zone. When not specified, the first availability zone from the query is used.
- **vpc_id**: Assigned by referencing the id of the created VPC resource huaweicloud_vpc.test.
- **subnet_id**: Assigned by referencing the id of the created subnet resource huaweicloud_vpc_subnet.test.
- **security_group_id**: Assigned by referencing the id of the created security group resource huaweicloud_networking_secgroup.test.
- **mode**: Assigned by referencing the input variable instance_mode, specifying the instance type.
- **datastore**: Configures the database type, version, and storage engine, assigned by referencing the input variables database_type, database_version, and storage_engine respectively.
- **flavor**: Configures the node type, number, spec code, storage type, disk size, and node list to be deleted, assigned by referencing the input variables node_type, node_number, node_spec_code, node_storage_type, node_size, and node_list respectively.

### 8. Query DDS Instance Information

Add the following script to the TF file (such as main.tf) to query the created DDS instance information and obtain the primary node ID:

```hcl
# Query DDS instance information in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_dds_instances" "test" {
  name = huaweicloud_dds_instance.test.name

  depends_on = [huaweicloud_dds_instance.test]
}

locals {
  nodeId = try([for v in flatten(data.huaweicloud_dds_instances.test.instances[*].groups[*].nodes) : v if v.role == "Primary"][0].id, "")
}
```

**Parameter Description**:
- **name**: Assigned by referencing the name of the created DDS instance huaweicloud_dds_instance.test, specifying the instance name to query.
- **depends_on**: Explicitly depends on the DDS instance resource to ensure the instance is created before querying.
- **locals.nodeId**: Extracts the ID of the node with role Primary from the query results, used for subsequent EIP association.

### 9. Associate EIP with DDS Instance

Add the following script to the TF file (such as main.tf) to associate the EIP with the primary node of the DDS instance:

```hcl
# Associate the EIP with the DDS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_dds_instance_eip_associate" "test" {
  instance_id = huaweicloud_dds_instance.test.id
  node_id     = local.nodeId
  public_ip   = huaweicloud_vpc_eip.test.address
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the id of the created DDS instance huaweicloud_dds_instance.test, specifying the instance to associate.
- **node_id**: Assigned by referencing the local variable local.nodeId, specifying the node ID to associate.
- **public_ip**: Assigned by referencing the address of the created EIP resource huaweicloud_vpc_eip.test, specifying the public IP address to associate.

### 10. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a way to preset these configurations through `tfvars` files, avoiding repeated input during each execution.

Create a `terraform.tfvars` file in the working directory. The example content is as follows:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
vpc_name            = "example-vpc"
subnet_name         = "example-subnet"
security_group_name = "example-security-group"
eip_bandwidth_name  = "example-bandwidth"
instance_name       = "example-dds-instance"
```

**Usage**:

1. Save the above content as the `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 11. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DDS instance and associating the EIP
4. Run `terraform show` to view the created DDS instance and EIP association

## Reference Information

- [Huawei Cloud Document Database Service Product Documentation](https://support.huaweicloud.com/dds/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DDS Instance Associate EIP](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/dds-associate-eip)
