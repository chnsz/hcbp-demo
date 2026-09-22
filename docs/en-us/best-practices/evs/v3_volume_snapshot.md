# Deploy V3 Volume and Snapshot

## Application Scenario

Elastic Volume Service (EVS) is a high-performance, highly reliable, and scalable block storage service provided by Huawei Cloud, providing persistent storage for ECS instances. A snapshot is a complete copy of volume data at a specific point in time, which can be used for data backup and fast recovery, and is an important means of ensuring business data reliability.

This best practice will introduce how to use Terraform to automatically deploy an EVS V3 volume and snapshot, including automatic querying of availability zones and images, creation of the volume, and creation of a snapshot based on the volume.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Images (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [V3 Volume (huaweicloud_evsv3_volume)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evsv3_volume)
- [V3 Volume Snapshot (huaweicloud_evsv3_snapshot)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evsv3_snapshot)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_evsv3_volume

data.huaweicloud_images_images
    └── huaweicloud_evsv3_volume

huaweicloud_evsv3_volume
    └── huaweicloud_evsv3_snapshot
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration details.

### 2. Query the Availability Zones

Add the following script to the TF file (such as main.tf) to query the available availability zones in the current region:

```hcl
# Query the available availability zones in the current region
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- This data source requires no additional parameters and automatically queries the availability zone information in the current region. The returned `names` list can be used to specify the availability zone for the volume.

### 3. Query the Images

Add the following script to the TF file (such as main.tf) to query the images used to create the volume:

```hcl
# Query the images matching the criteria in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "volume_image_id" {
  description = "The ID of the image used to create the volume, if not specified, the first available image matching the criteria will be used"
  type        = string
  default     = ""
}

variable "volume_image_visibility" {
  description = "The visibility of the volume image"
  type        = string
  default     = "public"
}

variable "volume_image_os" {
  description = "The OS of the volume image"
  type        = string
  default     = "Ubuntu"
}

data "huaweicloud_images_images" "test" {
  count = var.volume_image_id == "" ? 1 : 0

  visibility = var.volume_image_visibility
  os         = var.volume_image_os
}
```

**Parameter Description**:
- **count**: The image list is queried only when the image is not specified through `volume_image_id`, assigned by referencing the input variable volume_image_id
- **visibility**: The visibility of the image, assigned by referencing the input variable volume_image_visibility
- **os**: The OS type of the image, assigned by referencing the input variable volume_image_os

### 4. Create a V3 Volume

Add the following script to the TF file (such as main.tf) to create a V3 volume:

```hcl
# Create a V3 volume in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "volume_type" {
  description = "The type of the volume"
  type        = string
  default     = "GPSSD"
}

variable "volume_availability_zone" {
  description = "The availability zone for the volume"
  type        = string
  default     = ""
  nullable    = false
}

variable "volume_description" {
  description = "The description of the volume"
  type        = string
  default     = ""
}

variable "volume_metadata" {
  description = "The metadata of the volume"
  type        = map(string)
  default     = {}
}

variable "volume_multiattach" {
  description = "The volume is shared volume or not"
  type        = bool
  default     = false
}

variable "volume_name" {
  description = "The name of the volume"
  type        = string
}

variable "volume_size" {
  description = "The size of the volume"
  type        = number
  default     = 40
}

variable "volume_tags" {
  description = "The tags of the volume"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_evsv3_volume" "test" {
  region            = var.region_name
  volume_type       = var.volume_type
  availability_zone = var.volume_availability_zone != "" ? var.volume_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  description       = var.volume_description
  image_id          = var.volume_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, "") : var.volume_image_id
  metadata          = var.volume_metadata
  multiattach       = var.volume_multiattach
  name              = var.volume_name
  size              = var.volume_size
  tags              = var.volume_tags
}
```

**Parameter Description**:
- **region**: The region where the resource is located, assigned by referencing the input variable region_name
- **volume_type**: The type of the volume, assigned by referencing the input variable volume_type
- **availability_zone**: The availability zone of the volume; if not specified, the first availability zone in the availability zone list is used
- **description**: The description of the volume, assigned by referencing the input variable volume_description
- **image_id**: The ID of the image used by the volume; if not specified, the first image ID in the image list is used
- **metadata**: The metadata of the volume, assigned by referencing the input variable volume_metadata
- **multiattach**: Whether the volume is a shared volume, assigned by referencing the input variable volume_multiattach
- **name**: The name of the volume, assigned by referencing the input variable volume_name
- **size**: The size of the volume, assigned by referencing the input variable volume_size
- **tags**: The tags of the volume, assigned by referencing the input variable volume_tags

### 5. Create a V3 Volume Snapshot

Add the following script to the TF file (such as main.tf) to create a snapshot based on the volume above:

```hcl
# Create a V3 volume snapshot in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "snapshot_name" {
  description = "The name of the snapshot"
  type        = string
}

variable "snapshot_metadata" {
  description = "The metadata information of the snapshot"
  type        = map(string)
  default     = {}
}

variable "snapshot_description" {
  description = "The description of the snapshot"
  type        = string
  default     = ""
}

resource "huaweicloud_evsv3_snapshot" "test" {
  region      = var.region_name
  volume_id   = huaweicloud_evsv3_volume.test.id
  name        = var.snapshot_name
  metadata    = var.snapshot_metadata
  description = var.snapshot_description
}
```

**Parameter Description**:
- **region**: The region where the resource is located, assigned by referencing the input variable region_name
- **volume_id**: The ID of the volume to which the snapshot belongs, assigned by referencing the ID of the volume created in the previous step
- **name**: The name of the snapshot, assigned by referencing the input variable snapshot_name
- **metadata**: The metadata of the snapshot, assigned by referencing the input variable snapshot_metadata
- **description**: The description of the snapshot, assigned by referencing the input variable snapshot_description

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
volume_type        = "GPSSD"
volume_description = "Created by terraform"
volume_name        = "tf_test_volume"

volume_metadata = {
  test = "terraform volume"
}

volume_tags = {
  foo = "bar"
  key = "value"
}

snapshot_name        = "tf_test_snapshot"
snapshot_description = "Created by terraform"

snapshot_metadata = {
  test = "terraform snapshot"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="volume_name=my-volume"`
2. Environment variables: `export TF_VAR_volume_name=my-volume`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the V3 volume and snapshot
4. Run `terraform show` to view the created V3 volume and snapshot

## Reference Information

- [Huawei Cloud EVS Product Documentation](https://support.huaweicloud.com/evs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EVS V3 Volume and Snapshot](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/evs/v3-volume-snapshot)
