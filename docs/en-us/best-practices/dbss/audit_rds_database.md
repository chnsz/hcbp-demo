# Deploy Audit RDS Database

## Application Scenario

Database Security Service (DBSS) provides database security audit capabilities with bypass deployment, supporting auditing of Huawei Cloud RDS databases. By creating an RDS instance and a DBSS audit instance, and adding the RDS database to the audit scope, you can record database access behaviors in real time, generate fine-grained audit reports, and receive alarms for risky and attack behaviors, helping enterprises meet compliance requirements and protect data assets. This best practice introduces how to use Terraform to automatically deploy a DBSS audit instance and add an RDS database, including creating VPC, subnet, security group, RDS instance, and DBSS instance, as well as configuring RDS database audit information.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS Flavors Data Source (data.huaweicloud_rds_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)
- [DBSS Flavors Data Source (data.huaweicloud_dbss_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dbss_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [RDS Instance Resource (huaweicloud_rds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DBSS Instance Resource (huaweicloud_dbss_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dbss_instance)
- [RDS Database Resource (huaweicloud_dbss_rds_database)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dbss_rds_database)

### Resource/Data Source Dependencies

```text
data.huaweicloud_availability_zones
    ├── huaweicloud_rds_instance
    │   └── huaweicloud_dbss_rds_database
    └── huaweicloud_dbss_instance
        └── huaweicloud_dbss_rds_database

data.huaweicloud_rds_flavors
    └── huaweicloud_rds_instance
        └── huaweicloud_dbss_rds_database

data.huaweicloud_dbss_flavors
    └── huaweicloud_dbss_instance
        └── huaweicloud_dbss_rds_database

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_rds_instance
        │   └── huaweicloud_dbss_rds_database
        └── huaweicloud_dbss_instance
            └── huaweicloud_dbss_rds_database

huaweicloud_networking_secgroup
    ├── huaweicloud_rds_instance
    │   └── huaweicloud_dbss_rds_database
    └── huaweicloud_dbss_instance
        └── huaweicloud_dbss_rds_database
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the configuration introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the RDS instance and DBSS instance:

```hcl
variable "availability_zone" {
  description = "The availability zone to which the DBSS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

# Query all availability zones in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the RDS instance and DBSS instance
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the availability zones data source. The data source is created only when `var.availability_zone` is empty.

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

# Create a VPC resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the RDS instance and DBSS instance
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

# Create a VPC subnet resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the RDS instance and DBSS instance
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

# Create a security group resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the RDS instance and DBSS instance
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable `security_group_name`
- **delete_default_rules**: Whether to delete default security group rules. Fixed to `true` in this practice

### 6. Query RDS Flavors

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the RDS instance:

```hcl
variable "rds_instance_flavor" {
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
  default     = "8.0"
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
  description = "The number of vCPUs of the RDS instance flavor"
  type        = number
  default     = 4
}

# Query matching RDS flavors in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the RDS instance
data "huaweicloud_rds_flavors" "test" {
  count = var.rds_instance_flavor == "" ? 1 : 0

  db_type       = var.database_type
  db_version    = var.database_version
  instance_mode = var.instance_mode
  group_type    = var.instance_group_type
  vcpus         = var.instance_flavor_vcpus
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the RDS flavors data source. The data source is created only when `var.rds_instance_flavor` is empty.
- **db_type**: Database type, assigned by referencing the input variable `database_type`
- **db_version**: Database version, assigned by referencing the input variable `database_version`
- **instance_mode**: Mode of the RDS instance, assigned by referencing the input variable `instance_mode`
- **group_type**: Performance specification type, assigned by referencing the input variable `instance_group_type`
- **vcpus**: Number of vCPUs of the RDS instance flavor, assigned by referencing the input variable `instance_flavor_vcpus`

### 7. Create RDS Instance

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an RDS instance resource:

```hcl
variable "rds_instance_name" {
  description = "The name of the RDS instance"
  type        = string
}

variable "volume_type" {
  description = "The type of the volume"
  type        = string
  default     = "CLOUDSSD"
}

variable "volume_size" {
  description = "The size of the volume in GB"
  type        = number
  default     = 100
}

# Create an RDS instance resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_rds_instance" "test" {
  name              = var.rds_instance_name
  availability_zone = var.availability_zone == "" ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : [var.availability_zone]
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  flavor            = var.rds_instance_flavor == "" ? try(data.huaweicloud_rds_flavors.test[0].flavors[0].name, null) : var.rds_instance_flavor

  db {
    type    = var.database_type
    version = var.database_version
  }

  volume {
    type = var.volume_type
    size = var.volume_size
  }
}
```

**Parameter Description**:
- **name**: RDS instance name, assigned by referencing the input variable `rds_instance_name`
- **availability_zone**: Availability zone list of the RDS instance. If the input variable is empty, it is assigned based on the availability zones data source result; otherwise, the input variable value is used
- **vpc_id**: ID of the VPC that the RDS instance belongs to, assigned by referencing the ID of the VPC resource
- **subnet_id**: ID of the subnet that the RDS instance belongs to, assigned by referencing the ID of the VPC subnet resource
- **security_group_id**: ID of the security group that the RDS instance belongs to, assigned by referencing the ID of the security group resource
- **flavor**: Flavor of the RDS instance. If the input variable is empty, it is assigned based on the RDS flavors data source result; otherwise, the input variable value is used
- **db**: Database configuration
  - **type**: Database type, assigned by referencing the input variable `database_type`
  - **version**: Database version, assigned by referencing the input variable `database_version`
- **volume**: Volume configuration
  - **type**: Volume type, assigned by referencing the input variable `volume_type`
  - **size**: Volume size in GB, assigned by referencing the input variable `volume_size`

### 8. Query DBSS Flavors

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the DBSS instance:

```hcl
variable "dbss_instance_flavor" {
  description = "The flavor ID of the DBSS instance"
  type        = string
  default     = ""
  nullable    = false
}

# Query all DBSS flavors in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the DBSS instance
data "huaweicloud_dbss_flavors" "test" {
  count = var.dbss_instance_flavor == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the DBSS flavors data source. The data source is created only when `var.dbss_instance_flavor` is empty.

### 9. Create DBSS Instance

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DBSS instance resource:

```hcl
variable "dbss_instance_name" {
  description = "The name of the DBSS instance"
  type        = string
}

variable "instance_spec_code" {
  description = "The spec code of the DBSS instance"
  type        = string
  default     = "dbss.bypassaudit.low"
}

variable "instance_description" {
  description = "The description of the DBSS instance"
  type        = string
  default     = ""
}

variable "instance_tags" {
  description = "The tags of the DBSS instance"
  type        = map(string)
  default     = {}
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = null
}

variable "charging_mode" {
  description = "The charging mode of the DBSS instance"
  type        = string
  default     = "prePaid"
}

variable "period_unit" {
  description = "The period unit of the DBSS instance"
  type        = string
  default     = "month"
}

variable "period" {
  description = "The period of the DBSS instance"
  type        = number
  default     = 1
}

variable "auto_renew" {
  description = "The auto renew of the DBSS instance"
  type        = string
  default     = "false"
}

locals {
  vpc_name      = var.vpc_name
  subnet_name   = var.subnet_name
  instance_name = var.dbss_instance_name

  product_spec_desc = jsonencode(
    {
      "specDesc" : {
        "zh-cn" : {
          "主机名称" : local.instance_name,
          "虚拟私有云" : local.vpc_name,
          "子网" : local.subnet_name
        },
        "en-us" : {
          "Instance Name" : local.instance_name,
          "VPC" : local.vpc_name,
          "Subnet" : local.subnet_name
        }
      }
    }
  )
}

# Create a DBSS instance resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_dbss_instance" "test" {
  name                  = var.dbss_instance_name
  availability_zone     = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  flavor                = var.dbss_instance_flavor == "" ? try(data.huaweicloud_dbss_flavors.test[0].flavors[0].id, null) : var.dbss_instance_flavor
  product_spec_desc     = local.product_spec_desc
  resource_spec_code    = var.instance_spec_code
  description           = var.instance_description
  tags                  = var.instance_tags
  enterprise_project_id = var.enterprise_project_id
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew
}
```

**Parameter Description**:
- **name**: DBSS instance name, assigned by referencing the input variable `dbss_instance_name`
- **availability_zone**: Availability zone of the DBSS instance. If the input variable is empty, it is assigned based on the availability zones data source result; otherwise, the input variable value is used
- **vpc_id**: ID of the VPC that the DBSS instance belongs to, assigned by referencing the ID of the VPC resource
- **subnet_id**: ID of the subnet that the DBSS instance belongs to, assigned by referencing the ID of the VPC subnet resource
- **security_group_id**: ID of the security group that the DBSS instance belongs to, assigned by referencing the ID of the security group resource
- **flavor**: Flavor ID of the DBSS instance. If the input variable is empty, it is assigned based on the DBSS flavors data source result; otherwise, the input variable value is used
- **product_spec_desc**: Product specification description, assigned by the local variable `product_spec_desc`
- **resource_spec_code**: Spec code of the DBSS instance, assigned by referencing the input variable `instance_spec_code`
- **description**: Description of the DBSS instance, assigned by referencing the input variable `instance_description`
- **tags**: Tags of the DBSS instance, assigned by referencing the input variable `instance_tags`
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable `enterprise_project_id`
- **charging_mode**: Charging mode of the DBSS instance, assigned by referencing the input variable `charging_mode`
- **period_unit**: Period unit of the DBSS instance, assigned by referencing the input variable `period_unit`
- **period**: Period of the DBSS instance, assigned by referencing the input variable `period`
- **auto_renew**: Whether to enable auto renew, assigned by referencing the input variable `auto_renew`

### 10. Add RDS Database

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an RDS database resource:

```hcl
# Create an RDS database resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_dbss_rds_database" "test" {
  instance_id = huaweicloud_dbss_instance.test.instance_id
  rds_id      = huaweicloud_rds_instance.test.id
  type        = upper(var.database_type)
}
```

**Parameter Description**:
- **instance_id**: DBSS instance ID, assigned by referencing `instance_id` of the DBSS instance resource
- **rds_id**: RDS instance ID, assigned by referencing the ID of the RDS instance resource
- **type**: Database type, assigned by converting the input variable `database_type` to uppercase

> Note: RDS database auditing depends on an existing DBSS instance and RDS instance. Ensure that network connectivity is available between them.

### 11. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables for configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory. Example content is as follows:

```hcl
# Network and security group configuration
vpc_name            = "tf_test_audit"
subnet_name         = "tf_test_audit"
security_group_name = "tf_test_audit"

# RDS instance configuration
rds_instance_name = "tf_test_audit"

# DBSS instance configuration
dbss_instance_name = "tf_test_audit"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; for other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="vpc_name=tf_test_audit" -var="rds_instance_name=tf_test_audit"`
2. Environment variables: `export TF_VAR_dbss_instance_name=tf_test_audit`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the audit RDS database resources
4. Run `terraform show` to view the created audit RDS database resources

## Reference Information

- [Huawei Cloud DBSS Product Documentation](https://support.huaweicloud.com/dbss/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DBSS Audit RDS Database Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dbss/audit-rds-database)
