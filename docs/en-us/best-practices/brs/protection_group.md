# Deploy Protection Group

## Application Scenario

Business Recovery Service (BRS) is a disaster recovery service for Elastic Cloud Server (ECS) and Elastic Volume Service (EVS). Through host-level replication, data redundancy, and cache acceleration technologies, it provides users with high-level data reliability and business continuity, known as Business Recovery Service.

Business Recovery Service helps protect business applications by replicating Elastic Cloud Server data and configuration information to disaster recovery sites, allowing business applications to start and run normally on disaster recovery site cloud servers during production site cloud server downtime, thereby improving business continuity.

Protection groups are the basic components in BRS disaster recovery solutions, used to define the scope and policies of disaster recovery protection. By creating protection groups, you can specify the availability zones of production sites and disaster recovery sites, establishing the basic infrastructure for disaster recovery protection. This best practice will introduce how to use Terraform to automatically deploy a BRS protection group environment.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [SDRS Domain Query Data Source (data.huaweicloud_sdrs_domain)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/sdrs_domain)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [SDRS Protection Group Resource (huaweicloud_sdrs_protection_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sdrs_protection_group)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_sdrs_protection_group

data.huaweicloud_sdrs_domain
    └── huaweicloud_sdrs_protection_group

huaweicloud_vpc
    └── huaweicloud_sdrs_protection_group
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../docs/introductions/prepare_before_deploy.md).

### 2. Create VPC Network

Add the following script to the TF file (such as main.tf) to inform Terraform to create VPC network resources:

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

# Create VPC resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing input variable vpc_cidr

### 3. Query Availability Zone Information Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create SDRS protection groups:

```hcl
# Get all availability zone information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating SDRS protection groups
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- No special parameters, gets all availability zone information in the current region

### 4. Query SDRS Domain Information Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create SDRS protection groups:

```hcl
# Get all SDRS domain information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating SDRS protection groups
data "huaweicloud_sdrs_domain" "test" {}
```

**Parameter Description**:
- No special parameters, gets all SDRS domain information in the current region

### 5. Create SDRS Protection Group

Add the following script to the TF file (such as main.tf) to inform Terraform to create SDRS protection group resources:

```hcl
variable "protection_group_name" {
  description = "Protection group name"
  type        = string
}

variable "source_availability_zone" {
  description = "Protection group production site availability zone"
  type        = string
  default     = ""

  validation {
    condition = (
      (var.source_availability_zone == "" && var.target_availability_zone == "") ||
      (var.source_availability_zone != "" && var.target_availability_zone != "")
    )
    error_message = "Both `source_availability_zone` and `target_availability_zone` must be set, or both must be empty"
  }
}

variable "target_availability_zone" {
  description = "Protection group disaster recovery site availability zone"
  type        = string
  default     = ""
}

variable "protection_group_dr_type" {
  description = "Deployment model"
  type        = string
  default     = null
}

variable "protection_group_description" {
  description = "Protection group description"
  type        = string
  default     = null
}

variable "protection_group_enable" {
  description = "Whether to enable protection group to start protection"
  type        = bool
  default     = null
}

# Create SDRS protection group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing)
resource "huaweicloud_sdrs_protection_group" "test" {
  name                     = var.protection_group_name
  source_availability_zone = var.source_availability_zone != "" ? var.source_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  target_availability_zone = var.target_availability_zone != "" ? var.target_availability_zone : try(data.huaweicloud_availability_zones.test.names[1], null)
  domain_id                = data.huaweicloud_sdrs_domain.test.id
  source_vpc_id            = huaweicloud_vpc.test.id
  dr_type                  = var.protection_group_dr_type
  description              = var.protection_group_description
  enable                   = var.protection_group_enable
}
```

**Parameter Description**:
- **name**: Protection group name, assigned by referencing input variable protection_group_name
- **source_availability_zone**: Protection group production site availability zone, assigned by referencing input variable source_availability_zone, uses the first availability zone if empty
- **target_availability_zone**: Protection group disaster recovery site availability zone, assigned by referencing input variable target_availability_zone, uses the second availability zone if empty
- **domain_id**: SDRS domain ID, assigned based on the return result of SDRS domain query data source
- **source_vpc_id**: Production site VPC ID, assigned based on the return result of VPC resource
- **dr_type**: Deployment model, assigned by referencing input variable protection_group_dr_type
- **description**: Protection group description, assigned by referencing input variable protection_group_description
- **enable**: Whether to enable protection group to start protection, assigned by referencing input variable protection_group_enable

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Basic resource information
vpc_name                     = "tf_test_sdrs_protection_group"
protection_group_name        = "tf_test_sdrs_protection_group"

# Network configuration
vpc_cidr = "192.168.0.0/16"

# Protection group configuration
protection_group_description = "Created by terraform script"
```

**Usage Method**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content in this `tfvars` file when executing terraform commands, other names need to add `.auto` definition before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="protection_group_name=my-group"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating BRS protection group
4. Run `terraform show` to view the created BRS protection group

## Reference Information

- [Huawei Cloud BRS Product Documentation](https://support.huaweicloud.com/sdrs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For BRS Protection Group](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sdrs/protection-group)
