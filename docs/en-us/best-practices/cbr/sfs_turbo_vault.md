# Deploy SFS Turbo Type Vault

## Application Scenario

Cloud Backup and Recovery (CBR) is a data protection service provided by Huawei Cloud, offering simple and easy-to-use backup services for both cloud and on-premises resources. When events such as virus intrusion, accidental deletion, or hardware/software failures occur, data can be restored to any backup point. SFS Turbo type vault is a type of vault in CBR service, specifically designed for backing up Scalable File Service (SFS Turbo) file systems.

SFS Turbo type vault supports complete backup of SFS Turbo file systems, ensuring that the entire file system environment can be quickly restored when failures occur. SFS Turbo is a high-performance file storage service provided by Huawei Cloud, specifically designed for high-performance computing and AI/ML workloads. Through CBR backup service, important file data security and recoverability can be ensured. This best practice will introduce how to use Terraform to automatically deploy a CBR SFS Turbo type vault, including creating SFS Turbo file systems, configuring backup policies, and creating vaults.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [SFS Turbo File System Resource (huaweicloud_sfs_turbo)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sfs_turbo)
- [CBR Backup Policy Resource (huaweicloud_cbr_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cbr_policy)
- [CBR Vault Resource (huaweicloud_cbr_vault)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cbr_vault)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_sfs_turbo.test
            └── huaweicloud_cbr_vault.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_sfs_turbo.test

huaweicloud_cbr_policy.test
    └── huaweicloud_cbr_vault.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zones Required for SFS Turbo File System Resource Creation via Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the result of which will be used to create the SFS Turbo file system:

```hcl
variable "availability_zone" {
  description = "Availability zone information for the SFS Turbo file system"
  type        = string
  default     = ""
}

# Get all availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create the SFS Turbo file system
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- No special parameters, gets all availability zone information in the current region

### 3. Create VPC

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

### 4. Create VPC Subnet

Add the following script to the TF file to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "Subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "Subnet CIDR block, if not specified, will calculate a subnet CIDR within the existing CIDR address block"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "Subnet gateway IP, if not specified, will calculate a gateway IP within the existing CIDR address block"
  type        = string
  default     = ""
}

# Create a VPC subnet resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  name              = var.subnet_name
  cidr              = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip        = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **vpc_id**: VPC ID, assigned by referencing the VPC resource (huaweicloud_vpc.test) ID
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block, prioritizes input variable, calculates using cidrsubnet function if empty
- **gateway_ip**: Gateway IP, prioritizes input variable, calculates using cidrhost function if empty
- **availability_zone**: Availability zone, prioritizes input variable, uses the first result from availability zone list query data source if empty

### 5. Create Security Group

Add the following script to the TF file to instruct Terraform to create a security group resource:

```hcl
variable "secgroup_name" {
  description = "Security group name"
  type        = string
}

# Create a security group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.secgroup_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable secgroup_name
- **delete_default_rules**: Whether to delete default rules, set to true to delete default security group rules

### 6. Create SFS Turbo File System

Add the following script to the TF file to instruct Terraform to create an SFS Turbo file system resource:

```hcl
variable "turbo_name" {
  description = "SFS Turbo file system name"
  type        = string
}

variable "turbo_size" {
  description = "SFS Turbo file system size (GB)"
  type        = number
}

# Create an SFS Turbo file system resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_sfs_turbo" "test" {
  name              = var.turbo_name
  size              = var.turbo_size
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **name**: SFS Turbo file system name, assigned by referencing the input variable turbo_name
- **size**: File system size, assigned by referencing the input variable turbo_size
- **vpc_id**: VPC ID, assigned by referencing the VPC resource (huaweicloud_vpc.test) ID
- **subnet_id**: Subnet ID, assigned by referencing the VPC subnet resource (huaweicloud_vpc_subnet.test) ID
- **security_group_id**: Security group ID, assigned by referencing the security group resource (huaweicloud_networking_secgroup.test) ID
- **availability_zone**: Availability zone, prioritizes input variable, uses the first result from availability zone list query data source if empty

### 7. Create CBR Backup Policy (Optional)

Add the following script to the TF file to instruct Terraform to create a CBR backup policy resource:

```hcl
variable "enable_policy" {
  description = "Whether to enable backup policy"
  type        = bool
  default     = false
}

# Create a CBR backup policy resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_cbr_policy" "test" {
  count = var.enable_policy ? 1 : 0

  name        = "${var.vault_name}-policy"
  type        = "backup"
  time_period = 20
  time_zone   = "UTC+08:00"
  enabled     = true

  backup_cycle {
    days            = "MO,TU"
    execution_times = ["06:00"]
  }
}
```

**Parameter Description**:
- **count**: Number of resource instances, used to control whether to create the backup policy resource, only creates when `var.enable_policy` is true
- **name**: Backup policy name, assigned by referencing the input variable vault_name and fixed suffix
- **type**: Policy type, set to "backup" for backup policy
- **time_period**: Backup retention time (days), set to 20 days
- **time_zone**: Time zone, set to "UTC+08:00"
- **enabled**: Whether to enable policy, set to true
- **backup_cycle.days**: Backup cycle, set to Monday and Tuesday
- **backup_cycle.execution_times**: Execution time, set to 06:00

### 8. Create CBR Vault

Add the following script to the TF file to instruct Terraform to create a CBR vault resource:

```hcl
variable "vault_name" {
  description = "CBR vault name"
  type        = string
}

variable "protection_type" {
  description = "Vault protection type (backup or replication)"
  type        = string
  default     = "backup"

  validation {
    condition     = contains(["backup", "replication"], var.protection_type)
    error_message = "Protection type must be 'backup' or 'replication'."
  }
}

variable "vault_size" {
  description = "CBR vault size (GB)"
  type        = number
}

variable "auto_expand" {
  description = "Whether to automatically expand when vault is full"
  type        = bool
  default     = false
}

variable "enterprise_project_id" {
  description = "Enterprise project ID to which the vault belongs"
  type        = string
  default     = "0"
}

variable "backup_name_prefix" {
  description = "Backup name prefix"
  type        = string
  default     = ""
}

variable "is_multi_az" {
  description = "Whether the vault is deployed across AZs"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Vault tags"
  type        = map(string)
  default = {
    environment = "test"
    terraform   = "true"
    service     = "sfs-turbo"
  }
}

# Create a CBR vault resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_cbr_vault" "test" {
  name                  = var.vault_name
  type                  = "turbo"
  protection_type       = var.protection_type
  size                  = var.vault_size
  auto_expand           = var.auto_expand
  enterprise_project_id = var.enterprise_project_id
  backup_name_prefix    = var.backup_name_prefix
  is_multi_az           = var.is_multi_az

  resources {
    includes = [
      huaweicloud_sfs_turbo.test.id
    ]
  }

  dynamic "policy" {
    for_each = var.enable_policy ? [1] : []
    content {
      id = huaweicloud_cbr_policy.test[0].id
    }
  }

  tags = var.tags
}
```

**Parameter Description**:
- **name**: Vault name, assigned by referencing the input variable vault_name
- **type**: Vault type, set to "turbo" for SFS Turbo type vault
- **protection_type**: Protection type, assigned by referencing the input variable protection_type
- **size**: Vault size, assigned by referencing the input variable vault_size
- **auto_expand**: Whether to auto-expand, assigned by referencing the input variable auto_expand
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **backup_name_prefix**: Backup name prefix, assigned by referencing the input variable backup_name_prefix
- **is_multi_az**: Whether to deploy across AZs, assigned by referencing the input variable is_multi_az
- **resources.includes**: List of included resource IDs, assigned by referencing the SFS Turbo file system resource (huaweicloud_sfs_turbo.test) ID
- **policy.id**: Policy ID, assigned by referencing the backup policy resource (huaweicloud_cbr_policy.test) ID when policy is enabled
- **tags**: Tags, assigned by referencing the input variable tags

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Network configuration
vpc_name    = "cbr-test-vpc"
subnet_name = "cbr-test-subnet"
secgroup_name = "cbr-test-sg"

# SFS Turbo file system configuration
turbo_name = "cbr-test-turbo"
turbo_size = 500

# CBR vault configuration
vault_name        = "cbr-vault-turbo"
vault_size        = 1000
enable_policy     = true
protection_type   = "backup"
auto_expand       = false
is_multi_az       = true

# Tag configuration
tags = {
  environment = "test"
  project     = "cbr-sfs-turbo-demo"
  terraform   = "true"
  service     = "sfs-turbo"
}
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

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the CBR SFS Turbo type vault
4. Run `terraform show` to view the details of the created CBR SFS Turbo type vault

## Reference Information

- [Huawei Cloud CBR Product Documentation](https://support.huaweicloud.com/cbr/index.html)
- [Huawei Cloud SFS Turbo Product Documentation](https://support.huaweicloud.com/sfs-turbo/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CBR Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cbr)
