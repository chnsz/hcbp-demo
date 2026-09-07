# Deploy Schema

## Application Scenario

Distributed Database Middleware (DDM) solves the capacity and performance bottlenecks of traditional databases through database and table sharding, enabling high-concurrency access to massive volumes of data. When business data continues to grow and a single database reaches its capacity or performance limit, you need to create a schema for the DDM instance to distribute data across multiple data nodes.

This best practice will introduce how to use Terraform to deploy a DDM schema, including creating a VPC, subnet, security group, deploying an RDS data node and a DDM instance, and creating a schema associated with the data node. Through this practice, you can quickly master the method of automatically deploying a DDM schema using Terraform, laying a solid foundation for subsequent data sharding management.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS Flavors (huaweicloud_rds_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)
- [DDM Engines (huaweicloud_ddm_engines)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ddm_engines)
- [DDM Flavors (huaweicloud_ddm_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ddm_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [RDS Instance (huaweicloud_rds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DDM Instance (huaweicloud_ddm_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ddm_instance)
- [DDM Schema (huaweicloud_ddm_schema)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ddm_schema)

### Resource/Data Source Dependencies

```
huaweicloud_availability_zones
    └── huaweicloud_rds_instance
    └── huaweicloud_ddm_instance
huaweicloud_rds_flavors
    └── huaweicloud_rds_instance
huaweicloud_ddm_engines
    └── huaweicloud_ddm_flavors
    └── huaweicloud_ddm_instance
huaweicloud_ddm_flavors
    └── huaweicloud_ddm_instance
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
    └── huaweicloud_rds_instance
    └── huaweicloud_ddm_instance
huaweicloud_vpc_subnet
    └── huaweicloud_rds_instance
    └── huaweicloud_ddm_instance
huaweicloud_networking_secgroup
    └── huaweicloud_rds_instance
    └── huaweicloud_ddm_instance
random_password
    └── huaweicloud_rds_instance
    └── huaweicloud_ddm_schema
huaweicloud_rds_instance
    └── huaweicloud_ddm_schema
huaweicloud_ddm_instance
    └── huaweicloud_ddm_schema
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the best practice script in the specified working directory, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script to the TF file (such as main.tf) to query the availability zones available for the DDM instance and RDS instance:

```hcl
# Query availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zones" {
  description = "The availability zones to which the DDM instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**Parameter description**:
- **availability_zones**: Assigned by referencing the input variable availability_zones. When no availability zone is specified, available availability zones are automatically queried.

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

**Parameter description**:
- **name**: Assigned by referencing the input variable vpc_name.
- **cidr**: Assigned by referencing the input variable vpc_cidr.

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

**Parameter description**:
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id.
- **name**: Assigned by referencing the input variable subnet_name.
- **cidr**: Assigned by referencing the input variable subnet_cidr. When not specified, it is automatically calculated based on the VPC CIDR.
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip. When not specified, it is automatically calculated.

### 5. Create Security Group

Add the following script to the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**Parameter description**:
- **name**: Assigned by referencing the input variable security_group_name.

### 6. Generate Random Password

Add the following script to the TF file (such as main.tf) to generate a random password when the RDS instance password is not specified:

```hcl
# Generate a random password in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "rds_instance_password" {
  description = "The password of the RDS instance"
  type        = string
  sensitive   = true
  default     = ""
  nullable    = false
}

resource "random_password" "test" {
  count = var.rds_instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "~!@#%^*-_+?"
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**Parameter description**:
- **count**: Creates the random password resource when the RDS instance password is not specified.
- **length**: The length of the password.
- **special**: Whether to include special characters.
- **override_special**: The set of special characters.
- **min_upper**: The minimum number of uppercase letters.
- **min_lower**: The minimum number of lowercase letters.
- **min_numeric**: The minimum number of numeric characters.
- **min_special**: The minimum number of special characters.

### 7. Query RDS Flavors

Add the following script to the TF file (such as main.tf) to query available RDS flavors when the RDS instance flavor is not specified:

```hcl
# Query RDS flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_flavor" {
  description = "The flavor of the RDS instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "database_type" {
  description = "The database type of the RDS instance"
  type        = string
  default     = "MySQL"
}

variable "database_version" {
  description = "The database version of the RDS instance"
  type        = string
  default     = "5.7"
}

variable "instance_mode" {
  description = "The mode of the RDS instance"
  type        = string
  default     = "single"
}

variable "instance_group_type" {
  description = "The performance specification"
  type        = string
  default     = "dedicated"
}

variable "instance_flavor_vcpus" {
  description = "The number of vCPUs for the RDS instance flavor"
  type        = number
  default     = 2
}

data "huaweicloud_rds_flavors" "test" {
  count = var.instance_flavor == "" ? 1 : 0

  db_type       = var.database_type
  db_version    = var.database_version
  instance_mode = var.instance_mode
  group_type    = var.instance_group_type
  vcpus         = var.instance_flavor_vcpus
}
```

**Parameter description**:
- **db_type**: Assigned by referencing the input variable database_type.
- **db_version**: Assigned by referencing the input variable database_version.
- **instance_mode**: Assigned by referencing the input variable instance_mode.
- **group_type**: Assigned by referencing the input variable instance_group_type.
- **vcpus**: Assigned by referencing the input variable instance_flavor_vcpus.

### 8. Create RDS Instance

Add the following script to the TF file (such as main.tf) to create an RDS instance as the data node of the DDM schema:

```hcl
# Create an RDS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "rds_instance_name" {
  description = "The name of the RDS instance"
  type        = string
}

variable "database_port" {
  description = "The port of the RDS instance"
  type        = number
  default     = 3306
}

variable "volume_type" {
  description = "The volume type of the RDS instance"
  type        = string
  default     = "CLOUDSSD"
}

variable "volume_size" {
  description = "The volume size of the RDS instance"
  type        = number
  default     = 40
}

resource "huaweicloud_rds_instance" "test" {
  name              = var.rds_instance_name
  availability_zone = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  flavor            = var.instance_flavor == "" ? try(data.huaweicloud_rds_flavors.test[0].flavors[0].name, null) : var.instance_flavor
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id

  db {
    type     = var.database_type
    version  = var.database_version
    port     = var.database_port
    password = var.rds_instance_password == "" ? try(random_password.test[0].result, null) : var.rds_instance_password
  }

  volume {
    type = var.volume_type
    size = var.volume_size
  }
}
```

**Parameter description**:
- **name**: Assigned by referencing the input variable rds_instance_name.
- **availability_zone**: Assigned by referencing the input variable availability_zones or data.huaweicloud_availability_zones.test.
- **flavor**: Assigned by referencing the input variable instance_flavor or data.huaweicloud_rds_flavors.test.
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id.
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id.
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id.
- **db.type**: Assigned by referencing the input variable database_type.
- **db.version**: Assigned by referencing the input variable database_version.
- **db.port**: Assigned by referencing the input variable database_port.
- **db.password**: Assigned by referencing the input variable rds_instance_password or random_password.test.
- **volume.type**: Assigned by referencing the input variable volume_type.
- **volume.size**: Assigned by referencing the input variable volume_size.

### 9. Query DDM Engines and Flavors

Add the following script to the TF file (such as main.tf) to query available DDM engines and flavors when the engine and flavor of the DDM instance are not specified:

```hcl
# Query DDM engines and flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_engine_id" {
  description = "The engine ID of the DDM instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_id" {
  description = "The flavor ID of the DDM instance"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_ddm_engines" "test" {
  count = var.instance_engine_id == "" ? 1 : 0
}

data "huaweicloud_ddm_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  engine_id = var.instance_engine_id == "" ? try(data.huaweicloud_ddm_engines.test[0].engines[0].id, null)  : var.instance_engine_id
}
```

**Parameter description**:
- **engine_id**: Assigned by referencing the input variable instance_engine_id or data.huaweicloud_ddm_engines.test.

### 10. Create DDM Instance

Add the following script to the TF file (such as main.tf) to create a DDM instance:

```hcl
# Create a DDM instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "ddm_instance_name" {
  description = "The name of the DDM instance"
  type        = string
}

variable "instance_node_num" {
  description = "The number of nodes in the DDM instance"
  type        = number
  default     = 2
}

variable "instance_parameters" {
  description = "The parameters of the DDM instance"

  type = list(object({
    name  = string
    value = string
  }))

  default = []
}

resource "huaweicloud_ddm_instance" "test" {
  name               = var.ddm_instance_name
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  engine_id          = var.instance_engine_id == "" ? try(data.huaweicloud_ddm_engines.test[0].engines[0].id, null)  : var.instance_engine_id
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_ddm_flavors.test[0].flavors[0].id, null)  : var.instance_flavor_id
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  node_num           = var.instance_node_num

  dynamic "parameters" {
    for_each = var.instance_parameters

    content {
      name  = parameters.value.name
      value = parameters.value.value
    }
  }
}
```

**Parameter description**:
- **name**: Assigned by referencing the input variable ddm_instance_name.
- **availability_zones**: Assigned by referencing the input variable availability_zones or data.huaweicloud_availability_zones.test.
- **engine_id**: Assigned by referencing the input variable instance_engine_id or data.huaweicloud_ddm_engines.test.
- **flavor_id**: Assigned by referencing the input variable instance_flavor_id or data.huaweicloud_ddm_flavors.test.
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id.
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id.
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id.
- **node_num**: Assigned by referencing the input variable instance_node_num.
- **parameters**: Assigned by referencing the input variable instance_parameters, multiple parameters can be configured.

### 11. Create DDM Schema

Add the following script to the TF file (such as main.tf) to create a DDM schema and associate it with the data node:

```hcl
# Create a DDM schema in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "schema_name" {
  description = "The name of the DDM schema"
  type        = string
}

variable "schema_shard_mode" {
  description = "The shard mode of the DDM schema"
  type        = string
  default     = "single"
}

variable "schema_shard_number" {
  description = "The number of shards in the same working mode"
  type        = number
  default     = 1
}

resource "huaweicloud_ddm_schema" "test" {
  instance_id  = huaweicloud_ddm_instance.test.id
  name         = var.schema_name
  shard_mode   = var.schema_shard_mode
  shard_number = var.schema_shard_number

  data_nodes {
    id             = huaweicloud_rds_instance.test.id
    admin_user     = "root"
    admin_password = var.rds_instance_password == "" ? try(random_password.test[0].result, null) : var.rds_instance_password
  }

  lifecycle {
    ignore_changes = [
      data_nodes,
    ]
  }
}
```

**Parameter description**:
- **instance_id**: Assigned by referencing huaweicloud_ddm_instance.test.id.
- **name**: Assigned by referencing the input variable schema_name.
- **shard_mode**: Assigned by referencing the input variable schema_shard_mode.
- **shard_number**: Assigned by referencing the input variable schema_shard_number.
- **data_nodes.id**: Assigned by referencing huaweicloud_rds_instance.test.id.
- **data_nodes.admin_user**: Fixed to root.
- **data_nodes.admin_password**: Assigned by referencing the input variable rds_instance_password or random_password.test.
- **lifecycle.ignore_changes**: Ignores changes to data_nodes to prevent the schema from being recreated due to data node changes.

### 12. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, avoiding repeated input during each execution.

Create a `terraform.tfvars` file in the working directory. The example content is as follows:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
vpc_name            = "example-vpc"
subnet_name         = "example-subnet"
security_group_name = "example-security-group"
rds_instance_name   = "example-rds-instance"
ddm_instance_name   = "example-ddm-instance"
schema_name         = "example-schema"
```

**Usage**:

1. Save the above content as the `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 13. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DDM schema
4. Run `terraform show` to view the created DDM schema

## Reference Information

- [Huawei Cloud DDM Product Documentation](https://support.huaweicloud.com/ddm/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DDM Schema](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ddm/ddm-schema)
