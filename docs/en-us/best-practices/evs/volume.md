# Deploy Cloud Volume

## Application Scenario

Elastic Volume Service (EVS) is a high-performance, highly reliable, and scalable block storage service provided by Huawei Cloud, providing persistent storage for ECS instances. EVS supports multiple storage types, including SSD, SAS, SATA, etc., meeting storage requirements for different business scenarios.

Cloud volumes are the core resources of the EVS service, providing persistent storage capabilities and supporting multiple storage types and performance configurations. Through cloud volumes, enterprises can provide reliable data storage for ECS instances, supporting advanced functions such as data backup, snapshots, and expansion. Cloud volumes support multiple device types and performance configurations, meeting storage requirements for different application scenarios. This best practice will introduce how to use Terraform to automatically deploy cloud volumes, including availability zone selection, storage type configuration, and performance parameter settings.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Cloud Volume Resource (huaweicloud_evs_volume)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_evs_volume.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zone Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the results of which are used to create cloud volumes:

```hcl
# Get all availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create cloud volumes
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- No additional parameters required, the data source will automatically get all availability zone information in the current region

### 3. Create Cloud Volume

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a cloud volume resource:

```hcl
variable "volume_availability_zone" {
  description = "Cloud volume availability zone"
  type        = string
  default     = ""
  nullable    = false
}

variable "volume_name" {
  description = "Cloud volume name"
  type        = string
}

variable "volume_type" {
  description = "Cloud volume type"
  type        = string
  default     = "SSD"
}

variable "voulme_size" {
  description = "Cloud volume size"
  type        = number
  default     = 40
}

variable "volume_description" {
  description = "Cloud volume description"
  type        = string
  default     = ""
}

variable "volume_multiattach" {
  description = "Whether the cloud volume is a shared disk"
  type        = bool
  default     = false
}

variable "volume_iops" {
  description = "Cloud volume IOPS"
  type        = number
  default     = null
}

variable "volume_throughput" {
  description = "Cloud volume throughput"
  type        = number
  default     = null
}

variable "volume_device_type" {
  description = "Cloud volume device type"
  type        = string
  default     = "VBD"
}

variable "enterprise_project_id" {
  description = "Cloud volume enterprise project ID"
  type        = string
  default     = null
}

variable "volume_tags" {
  description = "Cloud volume tags"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "Cloud volume charging mode"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "Cloud volume charging period unit"
  type        = string
  default     = null
}

variable "period" {
  description = "Cloud volume charging period"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "Whether the cloud volume auto-renews"
  type        = string
  default     = "false"
}

# Create a cloud volume resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_evs_volume" "test" {
  availability_zone     = var.volume_availability_zone != "" ? var.volume_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  name                  = var.volume_name
  volume_type           = var.volume_type
  size                  = var.voulme_size
  description           = var.volume_description
  multiattach           = var.volume_multiattach
  iops                  = var.volume_iops
  throughput            = var.volume_throughput
  device_type           = var.volume_device_type
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.volume_tags
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew
}
```

**Parameter Description**:
- **availability_zone**: Availability zone, prioritizes using input variable, uses first result from availability zone data source if empty
- **name**: Cloud volume name, assigned by referencing the input variable volume_name
- **volume_type**: Cloud volume type, assigned by referencing the input variable volume_type
- **size**: Cloud volume size, assigned by referencing the input variable voulme_size
- **description**: Cloud volume description, assigned by referencing the input variable volume_description
- **multiattach**: Whether it's a shared disk, assigned by referencing the input variable volume_multiattach
- **iops**: IOPS performance, assigned by referencing the input variable volume_iops
- **throughput**: Throughput performance, assigned by referencing the input variable volume_throughput
- **device_type**: Device type, assigned by referencing the input variable volume_device_type
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **tags**: Tags, assigned by referencing the input variable volume_tags
- **charging_mode**: Charging mode, assigned by referencing the input variable charging_mode
- **period_unit**: Charging period unit, assigned by referencing the input variable period_unit
- **period**: Charging period, assigned by referencing the input variable period
- **auto_renew**: Auto-renewal, assigned by referencing the input variable auto_renew

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Cloud volume configuration
volume_name        = "tf_test_vplume"
volume_type        = "SSD"
voulme_size        = 40
volume_description = "terraform test"
volume_device_type = "VBD"
volume_tags        = {
  foo = "bar"
  key = "value"
}
charging_mode      = "postPaid"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="volume_name=my-volume" -var="volume_type=SSD"`
2. Environment variables: `export TF_VAR_volume_name=my-volume`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating cloud volumes
4. Run `terraform show` to view the created cloud volumes

## Reference Information

- [Huawei Cloud EVS Product Documentation](https://support.huaweicloud.com/evs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EVS Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/evs)
