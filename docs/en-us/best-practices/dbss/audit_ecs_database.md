# Deploy Audit ECS Database

## Application Scenario

Database Security Service (DBSS) provides database security audit capabilities with bypass deployment, supporting auditing of Huawei Cloud RDS databases and self-built databases on ECS/BMS. By creating a DBSS audit instance and adding an ECS self-built database, you can record database access behaviors in real time, generate fine-grained audit reports, and receive alarms for risky and attack behaviors, helping enterprises meet compliance requirements and protect data assets. This best practice introduces how to use Terraform to automatically deploy a DBSS audit instance and add an ECS self-built database, including creating VPC, subnet, security group, and DBSS instance, as well as configuring self-built database audit information.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DBSS Flavors Data Source (data.huaweicloud_dbss_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dbss_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DBSS Instance Resource (huaweicloud_dbss_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dbss_instance)
- [ECS Database Resource (huaweicloud_dbss_ecs_database)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dbss_ecs_database)

### Resource/Data Source Dependencies

```text
data.huaweicloud_availability_zones
    └── huaweicloud_dbss_instance
        └── huaweicloud_dbss_ecs_database

data.huaweicloud_dbss_flavors
    └── huaweicloud_dbss_instance
        └── huaweicloud_dbss_ecs_database

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dbss_instance
            └── huaweicloud_dbss_ecs_database

huaweicloud_networking_secgroup
    └── huaweicloud_dbss_instance
        └── huaweicloud_dbss_ecs_database
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the configuration introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the DBSS instance:

```hcl
variable "availability_zone" {
  description = "The availability zone to which the DBSS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

# Query all availability zones in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the DBSS instance
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

# Create a VPC resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the DBSS instance
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

# Create a VPC subnet resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the DBSS instance
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

# Create a security group resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the DBSS instance
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable `security_group_name`
- **delete_default_rules**: Whether to delete default security group rules. Fixed to `true` in this practice

### 6. Query DBSS Flavors

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the DBSS instance:

```hcl
variable "instance_flavor" {
  description = "The flavor ID of the DBSS instance"
  type        = string
  default     = ""
  nullable    = false
}

# Query all DBSS flavors in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the DBSS instance
data "huaweicloud_dbss_flavors" "test" {
  count = var.instance_flavor == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the DBSS flavors data source. The data source is created only when `var.instance_flavor` is empty.

### 7. Create DBSS Instance

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DBSS instance resource:

```hcl
variable "instance_name" {
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
  instance_name = var.instance_name

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
  name                  = var.instance_name
  availability_zone     = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  flavor                = var.instance_flavor == "" ? try(data.huaweicloud_dbss_flavors.test[0].flavors[0].id, null) : var.instance_flavor
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
- **name**: DBSS instance name, assigned by referencing the input variable `instance_name`
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

### 8. Add ECS Self-Built Database

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an ECS self-built database resource:

```hcl
variable "database_name" {
  description = "The name of the self built database"
  type        = string
}

variable "database_type" {
  description = "The type of the self built database"
  type        = string
  default     = "MYSQL"
}

variable "database_version" {
  description = "The version of the self built database"
  type        = string
  default     = "8"
}

variable "database_ip_address" {
  description = "The IP address of the self built database"
  type        = string
}

variable "database_port" {
  description = "The port of the self built database"
  type        = string
  default     = "3306"
}

variable "database_os" {
  description = "The OS of the self built database"
  type        = string
  default     = "LINUX64"
}

variable "database_charset" {
  description = "The charset of the self built database"
  type        = string
  default     = "UTF8"
}

# Create an ECS self-built database resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_dbss_ecs_database" "test" {
  instance_id = huaweicloud_dbss_instance.test.instance_id
  name        = var.database_name
  type        = var.database_type
  version     = var.database_version
  ip          = var.database_ip_address
  port        = var.database_port
  os          = var.database_os
  charset     = var.database_charset
}
```

**Parameter Description**:
- **instance_id**: DBSS instance ID, assigned by referencing `instance_id` of the DBSS instance resource
- **name**: Name of the self-built database, assigned by referencing the input variable `database_name`
- **type**: Type of the self-built database, assigned by referencing the input variable `database_type`
- **version**: Version of the self-built database, assigned by referencing the input variable `database_version`
- **ip**: IP address of the self-built database, assigned by referencing the input variable `database_ip_address`
- **port**: Port of the self-built database, assigned by referencing the input variable `database_port`
- **os**: OS of the self-built database, assigned by referencing the input variable `database_os`
- **charset**: Charset of the self-built database, assigned by referencing the input variable `database_charset`

> Note: The ECS self-built database depends on an existing DBSS instance. Ensure that the target self-built database is network-reachable and fill in the database IP and other information according to the actual environment.

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables for configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory. Example content is as follows:

```hcl
# Network and security group configuration
vpc_name            = "tf_test_audit"
subnet_name         = "tf_test_audit"
security_group_name = "tf_test_audit"

# DBSS instance configuration
instance_name = "tf_test_audit"

# ECS self-built database configuration
database_name       = "tf_test_audit"
database_ip_address = "YourDatabaseIPAddress"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; for other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="vpc_name=tf_test_audit" -var="database_ip_address=YourDatabaseIPAddress"`
2. Environment variables: `export TF_VAR_instance_name=tf_test_audit`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the audit ECS database resources
4. Run `terraform show` to view the created audit ECS database resources

## Reference Information

- [Huawei Cloud DBSS Product Documentation](https://support.huaweicloud.com/dbss/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DBSS Audit ECS Database Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dbss/audit-ecs-database)
