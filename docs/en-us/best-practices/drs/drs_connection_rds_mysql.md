# Deploy RDS MySQL Connection

## Application Scenario

Data Replication Service (DRS) is a one-stop data replication service provided by Huawei Cloud, dedicated to solving data flow problems in scenarios such as database cloud migration, database migration, real-time database synchronization, and database disaster recovery. Before using DRS to carry out migration, synchronization, or disaster recovery tasks, you need to maintain the access information of source and target databases through the connection management function, including database type, IP address and port, username and password, SSL configuration, and driver configuration.

This best practice will introduce how to use Terraform to automatically create a DRS connection for an RDS MySQL instance, including the creation of VPC, subnet, security group, RDS MySQL instance, and DRS connection, helping you quickly establish the connection between DRS and the on-cloud MySQL database and laying a solid foundation for subsequent data replication tasks.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS Flavors (data.huaweicloud_rds_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)

### Resources

- [VPC (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RDS Instance (huaweicloud_rds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DRS Connection (huaweicloud_drs_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_connection)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
data.huaweicloud_rds_flavors.test
    └── huaweicloud_rds_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_rds_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_networking_secgroup_rule.test
        └── huaweicloud_rds_instance.test

huaweicloud_rds_instance.test
    └── huaweicloud_drs_connection.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a VPC

Add the following script in the TF file (such as main.tf) to create a VPC:

```hcl
# Create a VPC resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The VPC name"
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

### 3. Create a Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create a subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC resource created above
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The subnet CIDR block, assigned by referencing the input variable subnet_cidr; when the variable is empty, the subnet is automatically divided based on the VPC CIDR block
- **gateway_ip**: The subnet gateway IP, assigned by referencing the input variable gateway_ip; when the variable is empty, the gateway IP is automatically calculated based on the subnet CIDR block

### 4. Create a Security Group and Its Rules

Add the following script in the TF file (such as main.tf) to create a security group and its rules:

```hcl
# Create a security group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = 2

  security_group_id = huaweicloud_networking_secgroup.test.id
  ethertype         = "IPv4"
  remote_ip_prefix  = "192.168.0.0/16"
  protocol          = "tcp"
  direction         = count.index == 0 ? "ingress" : "egress"
  ports             = count.index == 0 ? "3306" : null
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group, set to true to keep only custom rules
- **security_group_id**: The ID of the security group to which the rule belongs, referencing the ID of the security group resource created above
- **ethertype**: The network type, set to IPv4
- **remote_ip_prefix**: The remote IP address range, set to 192.168.0.0/16
- **protocol**: The protocol type, set to tcp
- **direction**: The rule direction, the first is ingress and the second is egress
- **ports**: The port range, the ingress rule opens port 3306 for MySQL access

### 5. Query Availability Zones and RDS Flavors

Add the following script in the TF file (such as main.tf) to query availability zones and RDS flavors:

```hcl
# Query the availability zone list in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "rds_db_type" {
  description = "The database type for querying RDS flavors"
  type        = string
  default     = "MySQL"
}

variable "rds_db_version" {
  description = "The database version for querying RDS flavors"
  type        = string
  default     = "5.7"
}

variable "rds_instance_mode" {
  description = "The instance mode for querying RDS flavors"
  type        = string
  default     = "single"
}

data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_rds_flavors" "test" {
  db_type       = var.rds_db_type
  db_version    = var.rds_db_version
  instance_mode = var.rds_instance_mode
}
```

**Parameter Description**:
- **db_type**: The database type, assigned by referencing the input variable rds_db_type
- **db_version**: The database version, assigned by referencing the input variable rds_db_version
- **instance_mode**: The instance mode, assigned by referencing the input variable rds_instance_mode

### 6. Create an RDS MySQL Instance

Add the following script in the TF file (such as main.tf) to create an RDS MySQL instance:

```hcl
# Create an RDS instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "rds_name" {
  description = "The name of the RDS instance"
  type        = string
}

variable "rds_flavor" {
  description = "The flavor of the RDS instance. If not specified, it will be queried from data source"
  type        = string
  default     = ""
}

variable "rds_fixed_ip" {
  description = "The fixed IP address of the RDS instance"
  type        = string
  default     = "192.168.0.100"
}

variable "db_password" {
  description = "The password for the RDS root user and DRS connection"
  type        = string
  sensitive   = true
}

resource "huaweicloud_rds_instance" "test" {
  depends_on = [
    huaweicloud_networking_secgroup_rule.test,
  ]

  name              = var.rds_name
  flavor            = var.rds_flavor != "" ? var.rds_flavor : try(data.huaweicloud_rds_flavors.test.flavors[0].name, null)
  security_group_id = huaweicloud_networking_secgroup.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  vpc_id            = huaweicloud_vpc.test.id
  fixed_ip          = var.rds_fixed_ip

  availability_zone = [
    try(data.huaweicloud_availability_zones.test.names[0], ""),
  ]

  db {
    password = var.db_password
    type     = "MySQL"
    version  = "5.7"
    port     = 3306
  }

  volume {
    type = "CLOUDSSD"
    size = 40
  }
}
```

**Parameter Description**:
- **name**: The RDS instance name, assigned by referencing the input variable rds_name
- **flavor**: The instance flavor, assigned by referencing the input variable rds_flavor; when the variable is empty, it is queried from the RDS flavors data source
- **security_group_id**: The ID of the security group to which the instance belongs, referencing the ID of the security group resource created above
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet resource created above
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC resource created above
- **fixed_ip**: The fixed IP address of the instance, assigned by referencing the input variable rds_fixed_ip
- **availability_zone**: The availability zone where the instance is located, queried from the availability zones data source
- **db.password**: The database password, assigned by referencing the input variable db_password
- **db.type**: The database type, set to MySQL
- **db.version**: The database version, set to 5.7
- **db.port**: The database port, set to 3306
- **volume.type**: The volume type, set to CLOUDSSD
- **volume.size**: The volume size, set to 40GB

### 7. Create a DRS Connection

Add the following script in the TF file (such as main.tf) to create a DRS connection:

```hcl
# Create a DRS connection resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "connection_name" {
  description = "The DRS connection name"
  type        = string
}

variable "description" {
  description = "The description of the DRS connection"
  type        = string
  default     = ""
}

variable "db_port" {
  description = "The database port"
  type        = string
  default     = "3306"
}

variable "db_user" {
  description = "The database username"
  type        = string
  default     = "root"
}

variable "driver_name" {
  description = "The driver name of the connection configuration"
  type        = string
  default     = "mysql"
}

resource "huaweicloud_drs_connection" "test" {
  name        = var.connection_name
  db_type     = "mysql"
  description = var.description

  endpoint {
    endpoint_name = "cloud_mysql"
    instance_id   = huaweicloud_rds_instance.test.id
    db_port       = var.db_port
    db_user       = var.db_user
    db_password   = var.db_password
  }

  vpc {
    vpc_id            = huaweicloud_rds_instance.test.vpc_id
    subnet_id         = huaweicloud_rds_instance.test.subnet_id
    security_group_id = huaweicloud_networking_secgroup.test.id
  }

  ssl {
    ssl_link = false
  }

  config {
    driver_name = var.driver_name
  }

  lifecycle {
    ignore_changes = [
      endpoint.0.db_password,
    ]
  }
}
```

**Parameter Description**:
- **name**: The DRS connection name, assigned by referencing the input variable connection_name
- **db_type**: The database type, set to mysql
- **description**: The connection description, assigned by referencing the input variable description
- **endpoint.endpoint_name**: The connection endpoint name, set to cloud_mysql
- **endpoint.instance_id**: The ID of the instance corresponding to the connection endpoint, referencing the ID of the RDS instance resource created above
- **endpoint.db_port**: The database port, assigned by referencing the input variable db_port
- **endpoint.db_user**: The database username, assigned by referencing the input variable db_user
- **endpoint.db_password**: The database password, assigned by referencing the input variable db_password
- **vpc.vpc_id**: The ID of the VPC to which the connection belongs, referencing the VPC ID of the RDS instance
- **vpc.subnet_id**: The ID of the subnet to which the connection belongs, referencing the subnet ID of the RDS instance
- **vpc.security_group_id**: The ID of the security group to which the connection belongs, referencing the ID of the security group resource created above
- **ssl.ssl_link**: Whether to enable the SSL connection, set to false
- **config.driver_name**: The driver name of the connection configuration, assigned by referencing the input variable driver_name
- **lifecycle.ignore_changes**: Ignore changes to the endpoint.0.db_password attribute because it is not returned by the API

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# VPC and network variables
vpc_name            = "your_vpc"
subnet_name         = "your_subnet"
security_group_name = "your_security_group"

# RDS instance variables
rds_name    = "your_rds"
db_password = "TestDrs@123"

# DRS connection variables
connection_name = "your_drs_connection"
description     = "DRS connection for MySQL"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the RDS MySQL connection
4. Run `terraform show` to view the created RDS MySQL connection

## Reference Information

- [Huawei Cloud Data Replication Service Product Documentation](https://support.huaweicloud.com/drs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DRS RDS MySQL Connection](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/drs-connection-rds-mysql)
