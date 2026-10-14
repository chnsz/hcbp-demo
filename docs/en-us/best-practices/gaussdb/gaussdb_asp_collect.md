# Deploy ASP Collect

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud, supporting both centralized and distributed deployment modes. ASP (Application Support Package) collection gathers runtime snapshot data of a GaussDB instance within a specified time range and generates a downloadable analysis report, helping O&M engineers locate performance issues and analyze slow SQL statements and wait events.

This best practice will introduce how to use Terraform to automatically deploy GaussDB ASP collect, including VPC creation, subnet configuration, security group creation, GaussDB instance configuration, and ASP collect task creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Network ACL Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB Instance (huaweicloud_gaussdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB ASP Collect (huaweicloud_gaussdb_asp_collect)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_asp_collect)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_gaussdb_instance.test
        └── huaweicloud_gaussdb_asp_collect.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_gaussdb_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_gaussdb_instance.test

random_password.test
    └── huaweicloud_gaussdb_instance.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a Virtual Private Cloud

Add the following script in the TF file (such as main.tf):

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

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, using the default enterprise project when empty

### 3. Create a Virtual Private Cloud Subnet

Add the following script in the TF file (such as main.tf):

```hcl
# Create a VPC subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **vpc_id**: Assigned by referencing the ID of the VPC resource
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr, automatically calculated from the VPC CIDR when empty
- **gateway_ip**: Assigned by referencing the input variable gateway_ip, automatically calculated from the subnet CIDR when empty

### 4. Query the Availability Zones

Add the following script in the TF file (such as main.tf):

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- This data source requires no input parameters and is used to obtain the list of available zone names in the current region

### 5. Create a Security Group

Add the following script in the TF file (such as main.tf):

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
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable security_group_name
- **delete_default_rules**: Set to true to delete the default rules of the security group

### 6. Create a Random Password

Add the following script in the TF file (such as main.tf):

```hcl
# Create a random password resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_password" {
  description = "The password for the GaussDB instance"
  type        = string
  default     = ""
  sensitive   = true
}

resource "random_password" "test" {
  count = var.instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "!@%^*-_=+"
  min_lower        = 1
  min_upper        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**Parameter Description**:
- **count**: Creates a random password when the input variable instance_password is empty
- **length**: The password length is 12
- **special**: Allows special characters
- **override_special**: The set of allowed special characters
- **min_lower**, **min_upper**, **min_numeric**, **min_special**: Specify the minimum number of lowercase letters, uppercase letters, digits, and special characters respectively

### 7. Create a GaussDB Instance

Add the following script in the TF file (such as main.tf):

```hcl
# Create a GaussDB instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The GaussDB instance name"
  type        = string
}

variable "instance_volume_type" {
  description = "The storage volume type"
  type        = string
  default     = "ULTRAHIGH"
}

variable "instance_volume_size" {
  description = "The storage volume size in GB"
  type        = number
  default     = 40
}

resource "huaweicloud_gaussdb_instance" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  flavor            = "gaussdb.opengauss.ee.dn.m6.2xlarge.8.in"
  name              = var.instance_name
  password          = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  port              = "9000"
  sharding_num      = 1
  coordinator_num   = 2
  availability_zone = join(",", try(slice(data.huaweicloud_availability_zones.test.names, 0, 3), []))

  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  ha {
    mode             = "enterprise"
    replication_mode = "sync"
    consistency      = "eventual"
    instance_mode    = "enterprise"
  }

  volume {
    type = var.instance_volume_type
    size = var.instance_volume_size
  }
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing the ID of the VPC resource
- **subnet_id**: Assigned by referencing the ID of the VPC subnet resource
- **security_group_id**: Assigned by referencing the ID of the security group resource
- **flavor**: The instance flavor, fixed to `gaussdb.opengauss.ee.dn.m6.2xlarge.8.in`
- **name**: Assigned by referencing the input variable instance_name
- **password**: Assigned by referencing the input variable instance_password, using the random password when empty
- **port**: The database port, fixed to 9000
- **sharding_num**: The number of shards, fixed to 1
- **coordinator_num**: The number of coordinator nodes, fixed to 2
- **availability_zone**: Assigned by referencing the first three availability zones of the availability zones data source
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, using the default enterprise project when empty
- **ha.mode**: The high availability mode, fixed to `enterprise`
- **ha.replication_mode**: The replication mode, fixed to `sync`
- **ha.consistency**: The consistency mode, fixed to `eventual`
- **ha.instance_mode**: The instance mode, fixed to `enterprise`
- **volume.type**: Assigned by referencing the input variable instance_volume_type
- **volume.size**: Assigned by referencing the input variable instance_volume_size

### 8. Create an ASP Collect Task

Add the following script in the TF file (such as main.tf):

```hcl
# Create an ASP collect resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "start_time" {
  description = "The start time for ASP collect in yyyy-mm-ddThh:mm:ssZ format"
  type        = string
}

variable "end_time" {
  description = "The end time for ASP collect in yyyy-mm-ddThh:mm:ssZ format"
  type        = string
}

resource "huaweicloud_gaussdb_asp_collect" "test" {
  instance_id = huaweicloud_gaussdb_instance.test.id
  start_time  = var.start_time
  end_time    = var.end_time
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the ID of the GaussDB instance resource
- **start_time**: Assigned by referencing the input variable start_time, in `yyyy-mm-ddThh:mm:ssZ` format
- **end_time**: Assigned by referencing the input variable end_time, in `yyyy-mm-ddThh:mm:ssZ` format

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
instance_name       = "tf_test_gaussdb_instance"
start_time          = "2026-08-24T09:00:00+0000"
end_time            = "2026-08-24T12:00:00+0000"
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

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the ASP collect task
4. Run `terraform show` to view the created ASP collect task

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB ASP Collect](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-asp-collect)
