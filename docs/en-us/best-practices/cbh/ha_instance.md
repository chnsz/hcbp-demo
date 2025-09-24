# Deploy Cloud Bastion Host HA Instance

## Application Scenario

Cloud Bastion Host (CBH) is a security operation and maintenance management service provided by Huawei Cloud, offering enterprises a unified security operation and maintenance entry point. CBH helps enterprises establish secure and compliant operation and maintenance management systems through centralized identity authentication, permission management, and operation auditing.

Cloud Bastion Host HA instance provides high availability assurance, ensuring service continuity and reliability through master-standby architecture. HA instance supports cross-availability zone deployment. When the master instance fails, the standby instance can automatically take over the service, ensuring business continuity. Before creation, you need to confirm the CBH specification type, master-standby availability zone configuration, network parameters, and security group rules based on actual application scenarios.

This best practice will introduce how to use Terraform to automatically deploy a cloud bastion host HA instance.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_cbh_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cbh_availability_zones)
- [Cloud Bastion Host Flavors Query Data Source (data.huaweicloud_cbh_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cbh_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Cloud Bastion Host HA Instance Resource (huaweicloud_cbh_ha_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cbh_ha_instance)

### Resource/Data Source Dependencies

```
data.huaweicloud_cbh_availability_zones
    └── huaweicloud_cbh_ha_instance

data.huaweicloud_cbh_flavors
    └── huaweicloud_cbh_ha_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_cbh_ha_instance

huaweicloud_networking_secgroup
    └── huaweicloud_cbh_ha_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zones Required for Cloud Bastion Host HA Instance Resource Creation via Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the result of which will be used to create the cloud bastion host HA instance:

```hcl
variable "master_availability_zone" {
  description = "Availability zone information for the master instance of the cloud bastion host HA instance"
  type        = string
  default     = ""
}

variable "slave_availability_zone" {
  description = "Availability zone information for the slave instance of the cloud bastion host HA instance"
  type        = string
  default     = ""
}

# Get all availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create the cloud bastion host HA instance
data "huaweicloud_cbh_availability_zones" "test" {
  count = var.master_availability_zone == "" || var.slave_availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the availability zone list query data source, only creates the data source when `var.master_availability_zone` or `var.slave_availability_zone` is empty (i.e., executes availability zone list query)

### 3. Query Specification Information Required for Cloud Bastion Host HA Instance Resource Creation via Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the result of which will be used to create the cloud bastion host HA instance:

```hcl
variable "instance_flavor_id" {
  description = "Specification ID for the cloud bastion host HA instance"
  type        = string
  default     = ""
}

variable "instance_flavor_type" {
  description = "Specification type for the cloud bastion host HA instance"
  type        = string
  default     = "basic"
}

# Get all cloud bastion host specification information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create the cloud bastion host HA instance
data "huaweicloud_cbh_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0
  type  = var.instance_flavor_type
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the specification list query data source, only creates the data source when `var.instance_flavor_id` is empty (i.e., executes specification list query)
- **type**: Specification type for the cloud bastion host HA instance, assigned by referencing the input variable instance_flavor_type

### 4. Create VPC

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  description = "VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "192.168.0.0/16"
}

# Create a VPC resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable vpc_cidr

### 5. Create VPC Subnet

Add the following script to the TF file to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "Subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "Subnet CIDR block"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "Subnet gateway IP"
  type        = string
  default     = ""
}

# Create a VPC subnet resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: VPC ID to which the subnet belongs, assigned by referencing the VPC resource (huaweicloud_vpc) ID
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block, automatically calculated when subnet_cidr is empty, otherwise uses the input variable subnet_cidr value
- **gateway_ip**: Subnet gateway IP, automatically calculated when subnet_gateway_ip is empty, otherwise uses the input variable subnet_gateway_ip value

### 6. Create Security Group

Add the following script to the TF file to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "Security group name"
  type        = string
}

# Create a security group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to true to delete default rules

### 7. Create Cloud Bastion Host HA Instance

Add the following script to the TF file to instruct Terraform to create a cloud bastion host HA instance resource:

```hcl
variable "instance_name" {
  description = "Cloud bastion host HA instance name"
  type        = string
}

variable "instance_password" {
  description = "Cloud bastion host HA instance login password"
  type        = string
  sensitive   = true
}

variable "charging_mode" {
  description = "Cloud bastion host HA instance billing mode"
  type        = string
  default     = "prePaid"
}

variable "period_unit" {
  description = "Cloud bastion host HA instance billing period unit"
  type        = string
  default     = "month"
}

variable "period" {
  description = "Cloud bastion host HA instance billing period"
  type        = number
  default     = 1
}

variable "auto_renew" {
  description = "Whether to enable auto-renewal for the cloud bastion host HA instance"
  type        = string
  default     = "false"
}

# Create a cloud bastion host HA instance resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_cbh_ha_instance" "test" {
  name                     = var.instance_name
  flavor_id                = var.instance_flavor_id == "" ? try(data.huaweicloud_cbh_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  vpc_id                   = huaweicloud_vpc.test.id
  subnet_id                = huaweicloud_vpc_subnet.test.id
  security_group_id        = huaweicloud_networking_secgroup.test.id
  master_availability_zone = var.master_availability_zone == "" ? try(data.huaweicloud_cbh_availability_zones.test[0].availability_zones[0].name, null) : var.master_availability_zone
  slave_availability_zone  = var.slave_availability_zone == "" ? try(data.huaweicloud_cbh_availability_zones.test[0].availability_zones[1].name, null) : var.slave_availability_zone
  password                 = var.instance_password
  charging_mode            = var.charging_mode
  period_unit              = var.period_unit
  period                   = var.period
  auto_renew               = var.auto_renew

  # If you want to change some of the following parameters, you need to remove the corresponding fields from "lifecycle.ignore_changes".
  lifecycle {
    ignore_changes = [
      flavor_id,
      master_availability_zone,
      slave_availability_zone,
    ]
  }
}
```

**Parameter Description**:
- **name**: Cloud bastion host HA instance name, assigned by referencing the input variable instance_name
- **flavor_id**: Cloud bastion host HA instance specification ID, when instance_flavor_id is empty, assigned based on the specification list query data source (data.huaweicloud_cbh_flavors) return result, otherwise uses the input variable instance_flavor_id value
- **vpc_id**: VPC ID to which the cloud bastion host HA instance belongs, assigned by referencing the VPC resource (huaweicloud_vpc) ID
- **subnet_id**: Subnet ID to which the cloud bastion host HA instance belongs, assigned by referencing the VPC subnet resource (huaweicloud_vpc_subnet) ID
- **security_group_id**: Security group ID to which the cloud bastion host HA instance belongs, assigned by referencing the security group resource (huaweicloud_networking_secgroup) ID
- **master_availability_zone**: Availability zone where the master instance of the cloud bastion host HA instance is located, when master_availability_zone is empty, assigned based on the availability zone list query data source (data.huaweicloud_cbh_availability_zones) return result, otherwise uses the input variable master_availability_zone value
- **slave_availability_zone**: Availability zone where the slave instance of the cloud bastion host HA instance is located, when slave_availability_zone is empty, assigned based on the availability zone list query data source (data.huaweicloud_cbh_availability_zones) return result, otherwise uses the input variable slave_availability_zone value
- **password**: Cloud bastion host HA instance login password, assigned by referencing the input variable instance_password
- **charging_mode**: Cloud bastion host HA instance billing mode, assigned by referencing the input variable charging_mode
- **period_unit**: Cloud bastion host HA instance billing period unit, assigned by referencing the input variable period_unit
- **period**: Cloud bastion host HA instance billing period, assigned by referencing the input variable period
- **auto_renew**: Whether to enable auto-renewal for the cloud bastion host HA instance, assigned by referencing the input variable auto_renew

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Basic resource information
vpc_name            = "tf_test_cbh_ha_vpc"
subnet_name         = "tf_test_cbh_ha_subnet"
security_group_name = "tf_test_cbh_ha_sg"
instance_name       = "tf_test_cbh_ha_instance"
instance_password   = "YourCBHHAInstancePassword!"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the cloud bastion host HA instance
4. Run `terraform show` to view the details of the created cloud bastion host HA instance

## Reference Information

- [Huawei Cloud Cloud Bastion Host Product Documentation](https://support.huaweicloud.com/cbh/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Cloud Bastion Host Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cbh)
