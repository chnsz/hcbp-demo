# Deploy Schema

## Application Scenario

Distributed Database Middleware (DDM) is a MySQL-compatible distributed relational database middleware that focuses on solving database distributed scaling issues, breaking through capacity and performance bottlenecks of traditional databases to enable highly concurrent access to massive volumes of data. A schema is the logical database unit exposed by DDM. By associating data nodes such as RDS and configuring the shard mode, you can implement database and table sharding and data routing. This best practice introduces how to use Terraform to automatically deploy a DDM schema, including creating VPC, subnet, and security group, deploying an RDS data node and a DDM instance, and creating a schema associated with the data node.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS Flavors Data Source (data.huaweicloud_rds_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)
- [DDM Engines Data Source (data.huaweicloud_ddm_engines)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ddm_engines)
- [DDM Flavors Data Source (data.huaweicloud_ddm_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ddm_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Random Password Resource (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [RDS Instance Resource (huaweicloud_rds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DDM Instance Resource (huaweicloud_ddm_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ddm_instance)
- [DDM Schema Resource (huaweicloud_ddm_schema)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ddm_schema)

### Resource/Data Source Dependencies

```text
data.huaweicloud_availability_zones
    ├── huaweicloud_rds_instance
    └── huaweicloud_ddm_instance

data.huaweicloud_rds_flavors
    └── huaweicloud_rds_instance

data.huaweicloud_ddm_engines
    ├── data.huaweicloud_ddm_flavors
    │   └── huaweicloud_ddm_instance
    └── huaweicloud_ddm_instance

data.huaweicloud_ddm_flavors
    └── huaweicloud_ddm_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_rds_instance
        └── huaweicloud_ddm_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_rds_instance
    └── huaweicloud_ddm_instance

random_password
    ├── huaweicloud_rds_instance
    └── huaweicloud_ddm_schema

huaweicloud_rds_instance
    └── huaweicloud_ddm_schema

huaweicloud_ddm_instance
    └── huaweicloud_ddm_schema
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the configuration introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the RDS instance and the DDM instance:

```hcl
variable "availability_zones" {
  description = "The availability zones to which the DDM instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

# Query all availability zones in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the RDS instance and the DDM instance
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the availability zones data source. The data source is created only when `var.availability_zones` is an empty list.

### 3. Create VPC Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

# Create a VPC resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the RDS instance and the DDM instance
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable `vpc_name`
- **cidr**: CIDR block of the VPC, assigned by referencing the input variable `vpc_cidr`

### 4. Create VPC Subnet Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC subnet resource:

```hcl
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

# Create a VPC subnet resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the RDS instance and the DDM instance
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: ID of the VPC that the subnet belongs to, assigned by referencing the ID of the VPC resource
- **name**: Subnet name, assigned by referencing the input variable `subnet_name`
- **cidr**: CIDR block of the subnet. If the input variable is empty, it is calculated automatically based on the VPC CIDR; otherwise, the input variable value is used
- **gateway_ip**: Gateway IP address of the subnet. If the input variable is empty, it is calculated automatically; otherwise, the input variable value is used

### 5. Create Security Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# Create a security group resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the RDS instance and the DDM instance
resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable `security_group_name`

### 6. Create Random Password

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a random password resource:

```hcl
variable "rds_instance_password" {
  description = "The password of the RDS instance"
  type        = string
  sensitive   = true
  default     = ""
  nullable    = false
}

# Create a random password resource, used to generate the database password when the RDS instance password is not specified
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

**Parameter Description**:
- **count**: Number of resource instances, used to control whether to create the random password resource. The resource is created only when `var.rds_instance_password` is empty
- **length**: Password length. Fixed to `12` in this practice
- **special**: Whether to include special characters. Fixed to `true` in this practice
- **override_special**: Allowed special character set. Fixed to `"~!@#%^*-_+?"` in this practice
- **min_upper**: Minimum number of uppercase letters in the password. Fixed to `1` in this practice
- **min_lower**: Minimum number of lowercase letters in the password. Fixed to `1` in this practice
- **min_numeric**: Minimum number of numeric characters in the password. Fixed to `1` in this practice
- **min_special**: Minimum number of special characters in the password. Fixed to `1` in this practice

### 7. Query RDS Flavors

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the RDS instance:

```hcl
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

# Query RDS flavors that match the conditions in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the RDS instance
data "huaweicloud_rds_flavors" "test" {
  count = var.instance_flavor == "" ? 1 : 0

  db_type       = var.database_type
  db_version    = var.database_version
  instance_mode = var.instance_mode
  group_type    = var.instance_group_type
  vcpus         = var.instance_flavor_vcpus
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the RDS flavors data source. The data source is created only when `var.instance_flavor` is empty
- **db_type**: Database type, assigned by referencing the input variable `database_type`
- **db_version**: Database version, assigned by referencing the input variable `database_version`
- **instance_mode**: RDS instance mode, assigned by referencing the input variable `instance_mode`
- **group_type**: Performance specification type, assigned by referencing the input variable `instance_group_type`
- **vcpus**: Number of vCPUs of the RDS flavor, assigned by referencing the input variable `instance_flavor_vcpus`

### 8. Create RDS Instance

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an RDS instance resource:

```hcl
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

# Create an RDS instance resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used as the data node of the DDM schema
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

**Parameter Description**:
- **name**: RDS instance name, assigned by referencing the input variable `rds_instance_name`
- **availability_zone**: Availability zone list of the RDS instance. If the input variable is an empty list, it is assigned based on the availability zones data source result; otherwise, the input variable value is used
- **flavor**: Flavor of the RDS instance. If the input variable is empty, it is assigned based on the RDS flavors data source result; otherwise, the input variable value is used
- **vpc_id**: ID of the VPC that the RDS instance belongs to, assigned by referencing the ID of the VPC resource
- **subnet_id**: ID of the subnet that the RDS instance belongs to, assigned by referencing the ID of the VPC subnet resource
- **security_group_id**: ID of the security group that the RDS instance belongs to, assigned by referencing the ID of the security group resource
- **db**: Database configuration
  - **type**: Database type, assigned by referencing the input variable `database_type`
  - **version**: Database version, assigned by referencing the input variable `database_version`
  - **port**: Database port, assigned by referencing the input variable `database_port`
  - **password**: Database password. If the input variable is empty, it is assigned based on the random password resource result; otherwise, the input variable value is used
- **volume**: Volume configuration
  - **type**: Volume type, assigned by referencing the input variable `volume_type`
  - **size**: Volume size, assigned by referencing the input variable `volume_size`

> Note: Keep sensitive information such as the database password secure and never commit it to version control.

### 9. Query DDM Engines

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the DDM instance and query flavors:

```hcl
variable "instance_engine_id" {
  description = "The engine ID of the DDM instance"
  type        = string
  default     = ""
  nullable    = false
}

# Query all DDM engines in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the DDM instance
data "huaweicloud_ddm_engines" "test" {
  count = var.instance_engine_id == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the DDM engines data source. The data source is created only when `var.instance_engine_id` is empty.

### 10. Query DDM Flavors

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the DDM instance:

```hcl
variable "instance_flavor_id" {
  description = "The flavor ID of the DDM instance"
  type        = string
  default     = ""
  nullable    = false
}

# Query DDM flavors of the specified engine in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the DDM instance
data "huaweicloud_ddm_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  engine_id = var.instance_engine_id == "" ? try(data.huaweicloud_ddm_engines.test[0].engines[0].id, null) : var.instance_engine_id
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the DDM flavors data source. The data source is created only when `var.instance_flavor_id` is empty.
- **engine_id**: DDM engine ID. If the input variable is empty, it is assigned based on the DDM engines data source result; otherwise, the input variable value is used

### 11. Create DDM Instance

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DDM instance resource:

```hcl
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

# Create a DDM instance resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_ddm_instance" "test" {
  name               = var.ddm_instance_name
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  engine_id          = var.instance_engine_id == "" ? try(data.huaweicloud_ddm_engines.test[0].engines[0].id, null) : var.instance_engine_id
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_ddm_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
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

**Parameter Description**:
- **name**: DDM instance name, assigned by referencing the input variable `ddm_instance_name`
- **availability_zones**: Availability zone list of the DDM instance. If the input variable is an empty list, it is assigned based on the availability zones data source result; otherwise, the input variable value is used
- **engine_id**: Engine ID of the DDM instance. If the input variable is empty, it is assigned based on the DDM engines data source result; otherwise, the input variable value is used
- **flavor_id**: Flavor ID of the DDM instance. If the input variable is empty, it is assigned based on the DDM flavors data source result; otherwise, the input variable value is used
- **vpc_id**: ID of the VPC that the DDM instance belongs to, assigned by referencing the ID of the VPC resource
- **subnet_id**: ID of the subnet that the DDM instance belongs to, assigned by referencing the ID of the VPC subnet resource
- **security_group_id**: ID of the security group that the DDM instance belongs to, assigned by referencing the ID of the security group resource
- **node_num**: Number of nodes in the DDM instance, assigned by referencing the input variable `instance_node_num`
- **parameters**: Parameter configuration of the DDM instance, created by the dynamic block `dynamic "parameters"` based on the input variable `instance_parameters`
  - **name**: Parameter name, assigned by referencing `name` in the input variable
  - **value**: Parameter value, assigned by referencing `value` in the input variable

### 12. Create DDM Schema

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DDM schema resource:

```hcl
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

# Create a DDM schema resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
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

**Parameter Description**:
- **instance_id**: ID of the DDM instance that the schema belongs to, assigned by referencing the ID of the DDM instance resource
- **name**: Schema name, assigned by referencing the input variable `schema_name`
- **shard_mode**: Shard mode of the schema, assigned by referencing the input variable `schema_shard_mode`
- **shard_number**: Number of shards in the same working mode, assigned by referencing the input variable `schema_shard_number`
- **data_nodes**: Data node configuration associated with the schema
  - **id**: Data node ID, assigned by referencing the ID of the RDS instance resource
  - **admin_user**: Administrator username of the data node. Fixed to `"root"` in this practice
  - **admin_password**: Administrator password of the data node. If the input variable is empty, it is assigned based on the random password resource result; otherwise, the input variable value is used
- **lifecycle.ignore_changes**: Lifecycle management that ignores changes to `data_nodes`

> Note: Keep sensitive information such as the data node administrator password secure and never commit it to version control.

### 13. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables for configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory. Example content is as follows:

```hcl
# Network and security group configuration
vpc_name            = "tf_test_schema"
subnet_name         = "tf_test_schema"
security_group_name = "tf_test_schema"

# RDS instance configuration
rds_instance_name = "tf_test_schema"

# DDM instance configuration
ddm_instance_name = "tf_test_schema"

# DDM schema configuration
schema_name = "tf_test_schema"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; for other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="vpc_name=tf_test_schema" -var="schema_name=tf_test_schema"`
2. Environment variables: `export TF_VAR_schema_name=tf_test_schema`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 14. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DDM schema
4. Run `terraform show` to view the created DDM schema

## Reference Information

- [Huawei Cloud DDM Product Documentation](https://support.huaweicloud.com/ddm/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DDM Schema Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ddm/ddm-schema)
