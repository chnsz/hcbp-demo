# Deploy Disk Snapshot

## Application Scenario

Elastic Volume Service (EVS) is a high-performance, highly reliable, and scalable block storage service provided by Huawei Cloud, providing persistent storage for ECS instances. EVS supports multiple storage types, including SSD, SAS, SATA, etc., meeting storage requirements for different business scenarios.

EVS disk snapshots are an important feature of the EVS service, used to create data backups of cloud volumes at specific points in time. Through disk snapshots, enterprises can protect important data, achieve rapid recovery, and support data migration and backup strategies. Disk snapshots support incremental backup, saving storage space and providing reliable data protection mechanisms. This best practice will introduce how to use Terraform to automatically deploy EVS disk snapshots, including cloud volume creation and snapshot configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Cloud Volume Resource (huaweicloud_evs_volume)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)
- [EVS Disk Snapshot Resource (huaweicloud_evs_snapshot)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_snapshot)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_evs_volume.test
        └── huaweicloud_evs_snapshot.test
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
  default     = "SAS"
}

variable "volume_size" {
  description = "Cloud volume size"
  type        = number
  default     = 20
}

variable "voluem_description" {
  description = "Cloud volume description"
  type        = string
  default     = ""
}

variable "vouleme_multiattach" {
  description = "Whether the cloud volume is a shared disk"
  type        = bool
  default     = false
}

# Create a cloud volume resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_evs_volume" "test" {
  availability_zone = var.volume_availability_zone != "" ? var.volume_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  name              = var.volume_name
  volume_type       = var.volume_type
  size              = var.volume_size
  description       = var.voluem_description
  multiattach       = var.vouleme_multiattach
}
```

**Parameter Description**:
- **availability_zone**: Availability zone, prioritizes using input variable, uses first result from availability zone data source if empty
- **name**: Cloud volume name, assigned by referencing the input variable volume_name
- **volume_type**: Cloud volume type, assigned by referencing the input variable volume_type
- **size**: Cloud volume size, assigned by referencing the input variable volume_size
- **description**: Cloud volume description, assigned by referencing the input variable voluem_description
- **multiattach**: Whether it's a shared disk, assigned by referencing the input variable vouleme_multiattach

### 4. Create EVS Disk Snapshot

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an EVS disk snapshot resource:

```hcl
variable "snapshot_name" {
  description = "Snapshot name"
  type        = string
}

variable "snapshot_description" {
  description = "Snapshot description"
  type        = string
  default     = ""
}

variable "snapshot_metadata" {
  description = "Snapshot metadata information"
  type        = map(string)
  default     = {}
}

variable "snapshot_force" {
  description = "Force create snapshot flag"
  type        = bool
  default     = false
}

# Create an EVS disk snapshot resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_evs_snapshot" "test" {
  volume_id   = huaweicloud_evs_volume.test.id
  name        = var.snapshot_name
  description = var.snapshot_description
  metadata    = var.snapshot_metadata
  force       = var.snapshot_force
}
```

**Parameter Description**:
- **volume_id**: Cloud volume ID, assigned by referencing the cloud volume resource (huaweicloud_evs_volume.test) ID
- **name**: Snapshot name, assigned by referencing the input variable snapshot_name
- **description**: Snapshot description, assigned by referencing the input variable snapshot_description
- **metadata**: Snapshot metadata, assigned by referencing the input variable snapshot_metadata
- **force**: Force create flag, assigned by referencing the input variable snapshot_force

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Cloud volume configuration
volume_name       = "tf_test_volume"
volume_type       = "SAS"
volume_size       = 20
voluem_description = "Test volume created by Terraform"

# Snapshot configuration
snapshot_name     = "tf_test_snapshot"
snapshot_description = "Test snapshot created by Terraform"
snapshot_metadata = {
  test = "terraform"
}
snapshot_force    = false
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="volume_name=my-volume" -var="snapshot_name=my-snapshot"`
2. Environment variables: `export TF_VAR_volume_name=my-volume`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating disk snapshots
4. Run `terraform show` to view the created disk snapshots

## Reference Information

- [Huawei Cloud EVS Product Documentation](https://support.huaweicloud.com/evs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EVS Disk Snapshot](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/evs/snapshot)
