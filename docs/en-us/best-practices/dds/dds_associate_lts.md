# Deploy DDS Instance Associate LTS

## Application Scenario

Document Database Service (DDS) is a high-performance, highly reliable, and secure distributed document database service provided by Huawei Cloud. It is fully compatible with the MongoDB protocol. In real business, it is necessary to centrally manage and analyze the audit logs of DDS instances to meet security compliance and O&M monitoring requirements.

This best practice will introduce how to use Terraform to create a DDS instance and associate it with an LTS log group and log stream, enabling the collection and storage of audit logs. Through this practice, you can learn how to use Terraform to automatically deploy DDS instances, VPC, security groups, and LTS resources, and connect the audit logs of the DDS instance to LTS, laying a foundation for subsequent log analysis and alarming.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DDS Flavors (data.huaweicloud_dds_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dds_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [LTS Log Group (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS Log Stream (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [DDS Instance (huaweicloud_dds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)
- [DDS LTS Log Association (huaweicloud_dds_lts_log)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_lts_log)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dds_instance
huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_dds_instance
huaweicloud_vpc_subnet
    └── huaweicloud_dds_instance
huaweicloud_networking_secgroup
    └── huaweicloud_dds_instance
data.huaweicloud_dds_flavors
    └── huaweicloud_dds_instance
huaweicloud_lts_group
    └── huaweicloud_lts_stream
huaweicloud_lts_stream
    └── huaweicloud_dds_lts_log
huaweicloud_dds_instance
    └── huaweicloud_dds_lts_log
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified working directory, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create VPC

Add the following script to the TF file (such as main.tf):

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
- **name**: Assigned by referencing the input variable vpc_name.
- **cidr**: Assigned by referencing the input variable vpc_cidr.

### 3. Create Subnet

Add the following script to the TF file (such as main.tf):

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
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id.
- **name**: Assigned by referencing the input variable subnet_name.
- **cidr**: Assigned by referencing the input variable subnet_cidr. When subnet_cidr is empty, it is automatically divided from the VPC CIDR.
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip. When subnet_gateway_ip is empty, the gateway address of the subnet is automatically calculated.

### 4. Create Security Group

Add the following script to the TF file (such as main.tf):

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
- **name**: Assigned by referencing the input variable security_group_name.
- **delete_default_rules**: Whether to delete default rules. Set to true to delete.

### 5. Create LTS Log Group and Log Stream

Add the following script to the TF file (such as main.tf):

```hcl
# Create an LTS log group and log stream in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "group_name" {
  description = "The name of the LTS log group"
  type        = string
}

variable "group_log_expiration_days" {
  description = "The log expiration time of the LTS log group"
  type        = number
  default     = 30
  nullable    = false
}

variable "stream_name" {
  description = "The name of the LTS log stream"
  type        = string
}

resource "huaweicloud_lts_group" "test" {
  group_name  = var.group_name
  ttl_in_days = var.group_log_expiration_days
}

resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.stream_name
}
```

**Parameter Description**:
- **huaweicloud_lts_group.test**:
  - **group_name**: Assigned by referencing the input variable group_name.
  - **ttl_in_days**: Assigned by referencing the input variable group_log_expiration_days, used to set the log retention time.
- **huaweicloud_lts_stream.test**:
  - **group_id**: Assigned by referencing huaweicloud_lts_group.test.id.
  - **stream_name**: Assigned by referencing the input variable stream_name.

### 6. Query Availability Zones and DDS Flavors

Add the following script to the TF file (such as main.tf):

```hcl
# Query availability zones and DDS flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
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
- **data.huaweicloud_availability_zones.test**: When availability_zone is empty, query the list of availability zones to automatically select one.
- **data.huaweicloud_dds_flavors.test**: When node_spec_code is empty, query DDS flavors based on engine name, vCPUs, memory, and node type to automatically select a flavor.

### 7. Create DDS Instance

Add the following script to the TF file (such as main.tf):

```hcl
# Create a DDS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone to which the DDS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

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

variable "instance_name" {
  description = "The name of the DDS instance"
  type        = string
}

variable "instance_mode" {
  description = "The mode of the DDS instance"
  type        = string
  default     = "ReplicaSet"
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
  mode              = var.instance_mode

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
- **name**: Assigned by referencing the input variable instance_name.
- **availability_zone**: Assigned by referencing the input variable availability_zone. When empty, the first availability zone is automatically selected.
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id.
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id.
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id.
- **mode**: Assigned by referencing the input variable instance_mode.
- **datastore**:
  - **type**: Assigned by referencing the input variable engine_name.
  - **version**: Assigned by referencing the input variable database_version.
  - **storage_engine**: Assigned by referencing the input variable storage_engine.
- **flavor**:
  - **type**: Assigned by referencing the input variable node_type.
  - **num**: Assigned by referencing the input variable node_number.
  - **spec_code**: Assigned by referencing the input variable node_spec_code. When empty, the queried flavor is automatically selected.
  - **storage**: Assigned by referencing the input variable node_storage_type.
  - **size**: Assigned by referencing the input variable node_size.
  - **node_list**: Assigned by referencing the input variable node_list.
- **port**: Assigned by referencing the input variable instance_port.
- **description**: Assigned by referencing the input variable instance_description.
- **password**: Assigned by referencing the input variable instance_password.
- **tags**: Assigned by referencing the input variable instance_tags.
- **charging_mode**: Assigned by referencing the input variable charging_mode.
- **period_unit**: Assigned by referencing the input variable period_unit.
- **period**: Assigned by referencing the input variable period.
- **auto_renew**: Assigned by referencing the input variable auto_renew.

### 8. Associate DDS Instance with LTS

Add the following script to the TF file (such as main.tf):

```hcl
# Associate the DDS instance with LTS in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "log_type" {
  description = "The log type"
  type        = string
  default     = "audit_log"
}

resource "huaweicloud_dds_lts_log" "test" {
  instance_id   = huaweicloud_dds_instance.test.id
  lts_group_id  = huaweicloud_lts_group.test.id
  lts_stream_id = huaweicloud_lts_stream.test.id
  log_type      = var.log_type
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing huaweicloud_dds_instance.test.id.
- **lts_group_id**: Assigned by referencing huaweicloud_lts_group.test.id.
- **lts_stream_id**: Assigned by referencing huaweicloud_lts_stream.test.id.
- **log_type**: Assigned by referencing the input variable log_type.

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this best practice, some resources and data sources use input variables to assign values to configurations. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, avoiding repeated input on each execution.

Create a `terraform.tfvars` file in the working directory. The example content is as follows:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
vpc_name            = "your_vpc_name"
subnet_name         = "your_subnet_name"
security_group_name = "your_security_group_name"
group_name          = "your_log_group_name"
stream_name         = "your_log_stream_name"
instance_name       = "your_instance_name"
```

**Usage**:

1. Save the above content as the `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`).
2. Modify the parameter values according to actual needs.
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file.

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment.
2. Run `terraform plan` to view the resource creation plan.
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DDS instance and associating it with LTS.
4. Run `terraform show` to view the created DDS instance and LTS association.

## Reference Information

- [Huawei Cloud Document Database Service Product Documentation](https://support.huaweicloud.com/dds/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DDS Instance Associate LTS](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/dds-associate-lts)
