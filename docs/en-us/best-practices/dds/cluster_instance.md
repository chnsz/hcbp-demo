# Deploy Cluster Instance

## Application Scenario

Document Database Service (DDS) is a high-performance, highly reliable, and secure distributed document database service provided by Huawei Cloud, fully compatible with the MongoDB protocol. A cluster instance (Sharding) implements data sharding and routing through three types of nodes: mongos, shard, and config, making it suitable for business scenarios with massive data storage and high concurrency access.

This best practice will introduce how to use Terraform to automatically deploy a DDS cluster instance, including the creation of VPC, subnet, and security group, as well as the configuration of mongos, shard, and config node flavors.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DDS Instance (huaweicloud_dds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
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
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

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
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: When the input variable subnet_cidr is empty, the subnet CIDR is automatically calculated based on the VPC CIDR, otherwise it is assigned by referencing the input variable subnet_cidr
- **gateway_ip**: When the input variable subnet_gateway_ip is empty, the gateway IP is automatically calculated based on the subnet CIDR, otherwise it is assigned by referencing the input variable subnet_gateway_ip

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

### 6. Create a DDS Cluster Instance

Add the following script in the TF file to create a DDS cluster instance:

```hcl
# Create a DDS cluster instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

variable "instance_flavors" {
  description = "The list of node flavor configurations for DDS instance"

  type = list(object({
    type      = string
    num       = number
    spec_code = string
    storage   = optional(string, "")
    size      = optional(number)
    node_list = optional(list(string), null)
  }))

  validation {
    condition     = length(var.instance_flavors) == 3
    error_message = "Create the DDS cluster instance, flavor configuration of the three type nodes must be specified"
  }
}

variable "instance_port" {
  description = "The database access port of the DDS instance"
  type        = number
  default     = 8635
}

variable "instance_password" {
  description = "The database access password of the DDS instance"
  sensitive   = true
  type        = string
  default     = ""
}

variable "instance_description" {
  description = "The description of the DDS instance"
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
  mode              = "Sharding"

  datastore {
    type           = var.database_type
    version        = var.database_version
    storage_engine = var.storage_engine
  }

  dynamic "flavor" {
    for_each = var.instance_flavors

    content {
      type      = flavor.value.type
      num       = flavor.value.num
      spec_code = flavor.value.spec_code
      storage   = flavor.value.storage
      size      = flavor.value.size
      node_list = flavor.value.node_list
    }
  }

  port          = var.instance_port
  password      = var.instance_password
  description   = var.instance_description
  tags          = var.instance_tags
  charging_mode = var.charging_mode
  period_unit   = var.period_unit
  period        = var.period
  auto_renew    = var.auto_renew
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable instance_name
- **availability_zone**: When the input variable availability_zone is empty, it references the first availability zone in the data source query result, otherwise it is assigned by referencing the input variable availability_zone
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id
- **mode**: Set to Sharding to create a cluster instance
- **datastore**: Database information, where type is assigned by referencing the input variable database_type, version is assigned by referencing the input variable database_version, and storage_engine is assigned by referencing the input variable storage_engine
- **flavor**: Node flavor configuration, assigned by iterating over the input variable instance_flavors through a dynamic block, including mongos, shard, and config nodes
- **port**: Assigned by referencing the input variable instance_port
- **password**: Assigned by referencing the input variable instance_password
- **description**: Assigned by referencing the input variable instance_description
- **tags**: Assigned by referencing the input variable instance_tags
- **charging_mode**: Assigned by referencing the input variable charging_mode
- **period_unit**: Assigned by referencing the input variable period_unit
- **period**: Assigned by referencing the input variable period
- **auto_renew**: Assigned by referencing the input variable auto_renew

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
vpc_name            = "tf_test_instance"
subnet_name         = "tf_test_instance"
security_group_name = "tf_test_instance"
instance_name       = "tf_test_instance"
instance_flavors    = [
  {
    type      = "mongos"
    num       = 2
    spec_code = "dds.mongodb.s6.large.2.mongos"
  },
  {
    type      = "shard"
    num       = 2
    spec_code = "dds.mongodb.s6.large.2.shard"
    storage   = "ULTRAHIGH"
    size      = 20
  },
  {
    type      = "config"
    num       = 1
    spec_code = "dds.mongodb.s6.large.2.config"
    storage   = "ULTRAHIGH"
    size      = 20
  }
]
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

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DDS cluster instance
4. Run `terraform show` to view the created DDS cluster instance

## Reference Information

- [Huawei Cloud Document Database Service Product Documentation](https://support.huaweicloud.com/dds/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DDS Cluster Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/dds-instance/cluster-instance)
