# Deploy DDS Instance Associate NAT

## Application Scenario

Document Database Service (DDS) is a high-performance, highly reliable, and secure distributed document database service provided by Huawei Cloud. It is fully compatible with the MongoDB protocol. When you need to allow a DDS instance to access the public network or provide services externally through a NAT gateway, you can use Terraform to automate the entire deployment process.

This best practice will introduce how to use Terraform to create a DDS instance and associate it with a NAT gateway, including creating a VPC, subnet, security group, NAT gateway, Elastic IP (EIP), and DDS instance, and finally using the `huaweicloud_dds_bind_gateway` resource to bind the DDS instance to the NAT gateway for secure and controllable public network access.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Querying Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Querying DDS Instance Information (data.huaweicloud_dds_instances)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dds_instances)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [NAT Gateway (huaweicloud_nat_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_gateway)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [DDS Instance (huaweicloud_dds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)
- [DDS Instance Associate NAT Gateway (huaweicloud_dds_bind_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_bind_gateway)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_dds_instance.test
huaweicloud_vpc.test
    ├── huaweicloud_vpc_subnet.test
    ├── huaweicloud_nat_gateway.test
    └── huaweicloud_dds_instance.test
huaweicloud_vpc_subnet.test
    ├── huaweicloud_nat_gateway.test
    └── huaweicloud_dds_instance.test
huaweicloud_networking_secgroup.test
    └── huaweicloud_dds_instance.test
huaweicloud_nat_gateway.test
    └── huaweicloud_dds_bind_gateway.test
huaweicloud_vpc_eip.test
    └── huaweicloud_dds_bind_gateway.test
huaweicloud_dds_instance.test
    ├── data.huaweicloud_dds_instances.test
    └── huaweicloud_dds_bind_gateway.test
data.huaweicloud_dds_instances.test
    └── huaweicloud_dds_bind_gateway.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified working directory, and ensure that it (or other TF files in the same level directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script to the TF file (such as main.tf) to automatically query available availability zones when no availability zone is specified:

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
- **availability_zone**: Assigned by referencing the input variable availability_zone. When this variable is empty, all availability zones will be queried.

### 3. Create VPC

Add the following script to the TF file (such as main.tf) to create a Virtual Private Cloud (VPC):

```hcl
# Create VPC in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **name**: Assigned by referencing the input variable vpc_name.
- **cidr**: Assigned by referencing the input variable vpc_cidr.

### 4. Create Subnet

Add the following script to the TF file (such as main.tf) to create a subnet:

```hcl
# Create subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id.
- **name**: Assigned by referencing the input variable subnet_name.
- **cidr**: Assigned by referencing the input variable subnet_cidr. When this variable is empty, it is automatically divided from the VPC CIDR.
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip. When this variable is empty, the gateway address of the subnet is automatically calculated.

### 5. Create Security Group

Add the following script to the TF file (such as main.tf) to create a security group:

```hcl
# Create security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **name**: Assigned by referencing the input variable security_group_name.
- **delete_default_rules**: Set to true to delete the default security group rules.

### 6. Create NAT Gateway

Add the following script to the TF file (such as main.tf) to create a NAT gateway:

```hcl
# Create NAT gateway in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "gateway_name" {
  description = "The name of the NAT gateway"
  type        = string
}

variable "gateway_spec" {
  description = "The specification of the NAT gateway"
  type        = string
  default     = "1"
}

resource "huaweicloud_nat_gateway" "test" {
  name      = var.gateway_name
  spec      = var.gateway_spec
  vpc_id    = huaweicloud_vpc.test.id
  subnet_id = huaweicloud_vpc_subnet.test.id
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable gateway_name.
- **spec**: Assigned by referencing the input variable gateway_spec.
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id.
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id.

### 7. Create Elastic IP

Add the following script to the TF file (such as main.tf) to create an Elastic IP (EIP):

```hcl
# Create EIP in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **publicip.type**: Assigned by referencing the input variable eip_type.
- **bandwidth.name**: Assigned by referencing the input variable eip_bandwidth_name.
- **bandwidth.share_type**: Fixed to "PER", indicating dedicated bandwidth.
- **bandwidth.size**: Assigned by referencing the input variable eip_bandwidth_size.
- **bandwidth.charge_mode**: Assigned by referencing the input variable eip_bandwidth_charge_mode.

### 8. Create DDS Instance

Add the following script to the TF file (such as main.tf) to create a DDS instance:

```hcl
# Create DDS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
  }
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable instance_name.
- **availability_zone**: Assigned by referencing the input variable availability_zone. When this variable is empty, the first availability zone from the query result is used.
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id.
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id.
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id.
- **mode**: Assigned by referencing the input variable instance_mode.
- **datastore.type**: Assigned by referencing the input variable database_type.
- **datastore.version**: Assigned by referencing the input variable database_version.
- **datastore.storage_engine**: Assigned by referencing the input variable storage_engine.
- **flavor.type**: Assigned by referencing the input variable node_type.
- **flavor.num**: Assigned by referencing the input variable node_number.
- **flavor.spec_code**: Assigned by referencing the input variable node_spec_code.
- **flavor.storage**: Assigned by referencing the input variable node_storage_type.
- **flavor.size**: Assigned by referencing the input variable node_size.

### 9. Query DDS Instance Information

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
- **name**: Assigned by referencing huaweicloud_dds_instance.test.name.
- **depends_on**: Explicitly depends on the DDS instance to ensure the instance is created before querying.
- **locals.nodeId**: Extracts the primary node ID from the query result for subsequent NAT gateway binding.

### 10. Associate NAT Gateway

Add the following script to the TF file (such as main.tf) to bind the primary node of the DDS instance to the NAT gateway:

```hcl
# Associate NAT gateway in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "external_service_port" {
  description = "The port of the EIP for providing services for external systems"
  type        = number
  default     = 8080
}

resource "huaweicloud_dds_bind_gateway" "test" {
  instance_id           = huaweicloud_dds_instance.test.id
  node_id               = local.nodeId
  nat_gateway_id        = huaweicloud_nat_gateway.test.id
  public_ip_id          = huaweicloud_vpc_eip.test.id
  external_service_port = var.external_service_port
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing huaweicloud_dds_instance.test.id.
- **node_id**: Assigned by referencing local.nodeId, which is the primary node ID of the DDS instance.
- **nat_gateway_id**: Assigned by referencing huaweicloud_nat_gateway.test.id.
- **public_ip_id**: Assigned by referencing huaweicloud_vpc_eip.test.id.
- **external_service_port**: Assigned by referencing the input variable external_service_port.

### 11. Preset Input Parameters Required for Resource Deployment (Optional)

In this best practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
vpc_name            = "your_vpc_name"
subnet_name         = "your_subnet_name"
security_group_name = "your_security_group_name"
gateway_name        = "your_nat_gateway_name"
eip_bandwidth_name  = "your_eip_bandwidth_name"
instance_name       = "your_instance_name"
```

**Usage**:

1. Save the above content as the `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DDS instance and associating it with the NAT gateway
4. Run `terraform show` to view the created DDS instance and NAT gateway binding relationship

## Reference Information

- [Huawei Cloud Document Database Service Product Documentation](https://support.huaweicloud.com/dds/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DDS Instance Associate NAT](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/dds-associate-nat)
