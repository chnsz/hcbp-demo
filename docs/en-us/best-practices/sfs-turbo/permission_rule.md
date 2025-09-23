# Deploy Permission Rules

## Application Scenario

Huawei Cloud Scalable File Service (SFS Turbo) permission rule functionality allows users to configure access control rules for SFS Turbo file systems, defining which IP addresses or network segments can access the file system and the permission levels for access. By configuring permission rules, you can implement secure access control, network isolation, and permission management for file systems.

This best practice is particularly suitable for scenarios that require controlling SFS Turbo file system access permissions, implementing network security isolation, managing multi-user access, such as multi-tenant environments, security-sensitive applications, enterprise file sharing, etc. This best practice will introduce how to use Terraform to automatically deploy SFS Turbo permission rules, including VPC network, security group, SFS Turbo file system, and permission rule creation, implementing a complete file system access control solution.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zone Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [SFS Turbo File System Resource (huaweicloud_sfs_turbo)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sfs_turbo)
- [SFS Turbo Permission Rule Resource (huaweicloud_sfs_turbo_perm_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sfs_turbo_perm_rule)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_sfs_turbo

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_sfs_turbo

huaweicloud_vpc_subnet
    └── huaweicloud_sfs_turbo

huaweicloud_networking_secgroup
    └── huaweicloud_sfs_turbo

huaweicloud_sfs_turbo
    └── huaweicloud_sfs_turbo_perm_rule
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zone Information

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query availability zone information:

```hcl
# Get all available availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create SFS Turbo file systems
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- This data source requires no additional parameters and automatically queries all availability zones in the current region

### 3. Create VPC Network

Add the following script to the TF file to instruct Terraform to create a VPC resource:

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

# Create a VPC resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable vpc_cidr, default value is "192.168.0.0/16"

### 4. Create VPC Subnet

Add the following script to the TF file to instruct Terraform to create a VPC subnet resource:

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

# Create a VPC subnet resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: VPC ID that the subnet belongs to, referencing the ID of the previously created VPC resource
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block, automatically calculated if subnet_cidr is empty, otherwise uses subnet_cidr value
- **gateway_ip**: Subnet gateway IP, automatically calculated if subnet_gateway_ip is empty, otherwise uses subnet_gateway_ip value

### 5. Create Security Group

Add the following script to the TF file to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "The name of the security group"
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
- **delete_default_rules**: Whether to delete default rules, set to true to delete default security group rules

### 6. Create SFS Turbo File System

Add the following script to the TF file to instruct Terraform to create an SFS Turbo file system resource:

```hcl
variable "turbo_availability_zone" {
  description = "The availability zone to which the SFS Turbo belongs"
  type        = string
  default     = ""
  nullable    = false
}

variable "turbo_name" {
  description = "The name of the SFS Turbo file system"
  type        = string
}

variable "turbo_size" {
  description = "The capacity of the SFS Turbo file system"
  type        = number
  default     = 500
}

variable "turbo_share_proto" {
  description = "The protocol of the SFS Turbo file system"
  type        = string
  default     = "NFS"
}

variable "turbo_share_type" {
  description = "The type of the SFS Turbo file system"
  type        = string
  default     = "STANDARD"
}

variable "turbo_hpc_bandwidth" {
  description = "The bandwidth specification of the SFS Turbo file system"
  type        = string
  default     = ""
}

# Create an SFS Turbo file system resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_sfs_turbo" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  availability_zone = var.turbo_availability_zone != "" ? var.turbo_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  name              = var.turbo_name
  size              = var.turbo_size
  share_proto       = var.turbo_share_proto
  share_type        = var.turbo_share_type
  hpc_bandwidth     = var.turbo_hpc_bandwidth
}
```

**Parameter Description**:
- **vpc_id**: VPC ID, referencing the ID of the previously created VPC resource
- **subnet_id**: Subnet ID, referencing the ID of the previously created VPC subnet resource
- **security_group_id**: Security group ID, referencing the ID of the previously created security group resource
- **availability_zone**: Availability zone, prioritizes using turbo_availability_zone variable, uses the first queried availability zone if empty
- **name**: SFS Turbo file system name, assigned by referencing the input variable turbo_name
- **size**: File system capacity, assigned by referencing the input variable turbo_size, default value is 500 (GB)
- **share_proto**: Sharing protocol, assigned by referencing the input variable turbo_share_proto, default value is "NFS"
- **share_type**: Sharing type, assigned by referencing the input variable turbo_share_type, default value is "STANDARD"
- **hpc_bandwidth**: HPC bandwidth specification, assigned by referencing the input variable turbo_hpc_bandwidth, default value is empty string

### 7. Create SFS Turbo Permission Rules

Add the following script to the TF file to instruct Terraform to create an SFS Turbo permission rule resource:

```hcl
variable "rule_ip_cidr" {
  description = "The IP address or network segment of the authorized object"
  type        = string
  default     = "192.168.0.0/16"
}

variable "rule_rw_type" {
  description = "The read and write permissions for the authorized object"
  type        = string
  default     = "rw"
}

variable "rule_user_type" {
  description = "The access permissions of the system users of the authorized object to the file system"
  type        = string
  default     = "no_root_squash"
}

# Create an SFS Turbo permission rule resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_sfs_turbo_perm_rule" "test" {
  share_id  = huaweicloud_sfs_turbo.test.id
  ip_cidr   = var.rule_ip_cidr
  rw_type   = var.rule_rw_type
  user_type = var.rule_user_type
}
```

**Parameter Description**:
- **share_id**: SFS Turbo file system ID, referencing the ID of the previously created SFS Turbo file system resource
- **ip_cidr**: IP address or network segment of the authorized object, assigned by referencing the input variable rule_ip_cidr, default value is "192.168.0.0/16"
- **rw_type**: Read and write permissions for the authorized object, assigned by referencing the input variable rule_rw_type, default value is "rw" (read-write permissions)
- **user_type**: Access permissions of the system users of the authorized object to the file system, assigned by referencing the input variable rule_user_type, default value is "no_root_squash" (no root user restrictions)

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC network configuration
vpc_name = "tf_test_perm_rule"

# Subnet configuration
subnet_name = "tf_test_perm_rule"

# Security group configuration
security_group_name = "tf_test_perm_rule"

# SFS Turbo file system configuration
turbo_name = "tf_test_perm_rule_demo"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="turbo_name=my-turbo"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating permission rules
4. Run `terraform show` to view the created permission rule details

## Reference Information

- [Huawei Cloud Scalable File Service Product Documentation](https://support.huaweicloud.com/sfs-turbo/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SFS Turbo Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sfs-turbo)
