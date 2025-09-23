# Deploy Disk Type Vault

## Application Scenario

Cloud Backup and Recovery (CBR) is a data protection service provided by Huawei Cloud, offering simple and easy-to-use backup services for both cloud and on-premises resources. When events such as virus intrusion, accidental deletion, or hardware/software failures occur, data can be restored to any backup point. Disk type vault is a type of vault in CBR service, specifically designed for backing up cloud disk (EVS) volumes.

Disk type vault supports complete backup of cloud disk volumes, ensuring that entire disk data can be quickly restored when failures occur. Cloud disk is a scalable virtual block storage service provided by Huawei Cloud, featuring high reliability, high performance, and easy scalability. Through CBR backup service, important disk data security and recoverability can be ensured. This best practice will introduce how to use Terraform to automatically deploy a CBR disk type vault, including creating cloud disk volumes and vaults.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Cloud Disk Volume Resource (huaweicloud_evs_volume)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)
- [CBR Vault Resource (huaweicloud_cbr_vault)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cbr_vault)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_evs_volume.test
        └── huaweicloud_cbr_vault.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zones Required for Cloud Disk Volume Resource Creation via Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the result of which will be used to create the cloud disk volume:

```hcl
variable "availability_zone" {
  description = "Availability zone information for the cloud disk volume"
  type        = string
  default     = ""
}

# Get all availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create the cloud disk volume
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- No special parameters, gets all availability zone information in the current region

### 3. Create Cloud Disk Volume

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a cloud disk volume resource:

```hcl
variable "volume_type" {
  description = "Cloud disk volume type"
  type        = string
}

variable "volume_name" {
  description = "Cloud disk volume name"
  type        = string
  default     = ""
}

variable "volume_size" {
  description = "Cloud disk volume size (GB)"
  type        = number
}

variable "volume_device_type" {
  description = "Cloud disk volume device type"
  type        = string
  default     = "VBD"
}

# Create a cloud disk volume resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_evs_volume" "test" {
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  volume_type       = var.volume_type
  name              = var.volume_name
  size              = var.volume_size
  device_type       = var.volume_device_type
}
```

**Parameter Description**:
- **availability_zone**: Availability zone, prioritizes input variable, uses the first result from availability zone list query data source if empty
- **volume_type**: Cloud disk volume type, assigned by referencing the input variable volume_type
- **name**: Cloud disk volume name, assigned by referencing the input variable volume_name
- **size**: Cloud disk volume size, assigned by referencing the input variable volume_size
- **device_type**: Cloud disk volume device type, assigned by referencing the input variable volume_device_type

### 4. Create CBR Vault

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CBR vault resource:

```hcl
variable "name" {
  description = "CBR vault name"
  type        = string
}

variable "type" {
  description = "CBR vault type"
  type        = string
  default     = "disk"
}

variable "protection_type" {
  description = "Vault protection type"
  type        = string
  default     = "backup"
}

variable "size" {
  description = "CBR vault size (GB)"
  type        = number
}

variable "enterprise_project_id" {
  description = "Enterprise project ID to which the vault belongs"
  type        = string
  default     = "0"
}

# Create a CBR vault resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_cbr_vault" "test" {
  name                  = var.name
  type                  = var.type
  protection_type       = var.protection_type
  size                  = var.size
  enterprise_project_id = var.enterprise_project_id

  resources {
    includes = [huaweicloud_evs_volume.test.id]
  }
}
```

**Parameter Description**:
- **name**: Vault name, assigned by referencing the input variable name
- **type**: Vault type, assigned by referencing the input variable type, defaults to "disk" for disk type vault
- **protection_type**: Protection type, assigned by referencing the input variable protection_type
- **size**: Vault size, assigned by referencing the input variable size
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **resources.includes**: List of included resource IDs, assigned by referencing the cloud disk volume resource (huaweicloud_evs_volume.test) ID

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Cloud disk volume configuration
volume_type = "SSD"
volume_size = 50
volume_name = "cbr-test-volume"
volume_device_type = "VBD"

# CBR vault configuration
name        = "tf_cbr_script"
size        = 100
type        = "disk"
protection_type = "backup"
enterprise_project_id = "0"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="volume_type=SSD" -var="volume_size=50"`
2. Environment variables: `export TF_VAR_volume_type=SSD`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the CBR disk type vault
4. Run `terraform show` to view the details of the created CBR disk type vault

## Reference Information

- [Huawei Cloud CBR Product Documentation](https://support.huaweicloud.com/cbr/index.html)
- [Huawei Cloud EVS Product Documentation](https://support.huaweicloud.com/evs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CBR Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cbr)
