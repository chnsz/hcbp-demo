# Deploy Backup Stop

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud, supporting both centralized and distributed deployment modes. During daily database O&M, backup tasks may run for a long time due to large data volumes or business peak hours, occupying instance resources and affecting business performance, so ongoing backup tasks need to be stopped in time.

This best practice will introduce how to use Terraform to automatically deploy a GaussDB instance and stop its ongoing backup, including VPC creation, subnet configuration, security group configuration, GaussDB instance creation, and backup stop operation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [GaussDB Flavors (data.huaweicloud_gaussdb_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB Instance (huaweicloud_gaussdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB Backup Stop (huaweicloud_gaussdb_backup_stop)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_backup_stop)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
data.huaweicloud_gaussdb_flavors.test
    └── huaweicloud_gaussdb_instance.test
        └── huaweicloud_gaussdb_backup_stop.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_gaussdb_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_networking_secgroup_rule.test
        └── huaweicloud_gaussdb_instance.test

random_password.test
    └── huaweicloud_gaussdb_instance.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the article [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

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
- **cidr**: Assigned by referencing the input variable vpc_cidr, defaulting to `192.168.0.0/16`
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; no enterprise project is specified when it is empty

### 3. Create a Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

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
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; automatically calculated from the VPC CIDR when empty
- **gateway_ip**: Assigned by referencing the input variable gateway_ip; automatically calculated from the VPC CIDR when empty

### 4. Query Availability Zones and GaussDB Flavors

Add the following script in the TF file (such as main.tf) to query availability zones and GaussDB flavors:

```hcl
# Query availability zones and GaussDB flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_gaussdb_flavors" "test" {
  version = "8.201"
  ha_mode = "centralization_standard"
}
```

**Parameter Description**:
- **version**: GaussDB version, fixed to `8.201`
- **ha_mode**: High availability mode, fixed to `centralization_standard`

### 5. Create a Security Group and Rules

Add the following script in the TF file (such as main.tf) to create a security group and rules:

```hcl
# Create a security group and security group rule resources in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  for_each = toset(["ingress", "egress"])

  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = each.key
  ethertype         = "IPv4"
  protocol          = "tcp"
  remote_ip_prefix  = "0.0.0.0/0"
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable security_group_name
- **delete_default_rules**: Deletes the default rules of the security group, fixed to `true`
- **direction**: Direction of the security group rule; both ingress and egress rules are created
- **ethertype**: IP protocol type, fixed to `IPv4`
- **protocol**: Protocol type, fixed to `tcp`
- **remote_ip_prefix**: Remote IP address range, fixed to `0.0.0.0/0`

### 6. Create a Random Password

Add the following script in the TF file (such as main.tf) to automatically generate a random password when no instance password is specified:

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
- **length**: Password length, fixed to `12`
- **special**: Whether to include special characters, fixed to `true`
- **override_special**: Set of allowed special characters
- **min_lower**, **min_upper**, **min_numeric**, **min_special**: Minimum number of characters of each type

### 7. Create a GaussDB Instance

Add the following script in the TF file (such as main.tf) to create a GaussDB instance:

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
  depends_on = [
    huaweicloud_networking_secgroup_rule.test
  ]

  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  flavor            = try(data.huaweicloud_gaussdb_flavors.test.flavors[0].spec_code, "")
  name              = var.instance_name
  password          = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  availability_zone = join(",", try(slice(data.huaweicloud_availability_zones.test.names, 0, 3), []))

  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  ha {
    mode             = "centralization_standard"
    replication_mode = "sync"
    consistency      = "eventual"
    instance_mode    = "basic"
  }

  volume {
    type = var.instance_volume_type
    size = var.instance_volume_size
  }
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id
- **subnet_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id
- **flavor**: Assigned by referencing data.huaweicloud_gaussdb_flavors.test.flavors[0].spec_code
- **name**: Assigned by referencing the input variable instance_name
- **password**: Assigned by referencing the input variable instance_password or the random password
- **availability_zone**: Assigned by referencing the availability zone list, taking the first three zones
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; no enterprise project is specified when it is empty
- **ha.mode**: High availability mode, fixed to `centralization_standard`
- **ha.replication_mode**: Replication mode, fixed to `sync`
- **ha.consistency**: Consistency mode, fixed to `eventual`
- **ha.instance_mode**: Instance mode, fixed to `basic`
- **volume.type**: Storage type, assigned by referencing the input variable instance_volume_type
- **volume.size**: Storage size, assigned by referencing the input variable instance_volume_size

### 8. Stop the GaussDB Instance Backup

Add the following script in the TF file (such as main.tf) to stop the ongoing backup of the GaussDB instance:

```hcl
# Stop the GaussDB instance backup in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_gaussdb_backup_stop" "test" {
  instance_id = huaweicloud_gaussdb_instance.test.id
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing huaweicloud_gaussdb_instance.test.id

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
instance_name       = "tf_test_gaussdb_instance"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB instance and stop its ongoing backup
4. Run `terraform show` to view the created GaussDB instance and the backup stop operation

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Backup Stop](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-instance-backup-stop)
