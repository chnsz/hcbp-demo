# Deploy Media Category and Media Asset

## Application Scenario

Video on Demand (VOD) is a one-stop video on-demand service provided by Huawei Cloud, supporting audio and video upload, transcoding, storage, acceleration and playback, helping enterprises and developers quickly build video on-demand services. Media categories are used to group massive audio and video resources, and a media asset is the core object that carries the audio and video source file in the VOD service.

This best practice will introduce how to use Terraform to automatically deploy a VOD media category and a media asset, including creating a media category, and creating a media asset by pulling the source file from a URL and associating it with the category.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Media Category (huaweicloud_vod_media_category)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_media_category)
- [Media Asset (huaweicloud_vod_media_asset)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_media_asset)

### Resource/Data Source Dependencies

```
huaweicloud_vod_media_category
    └── huaweicloud_vod_media_asset
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Media Category

Add the following script in the TF file (such as main.tf) to create a media category:

```hcl
# Create a media category resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "media_category_name" {
  description = "The name of the media category"
  type        = string
}

resource "huaweicloud_vod_media_category" "test" {
  name = var.media_category_name
}
```

**Parameter Description**:
- **name**: The name of the media category, assigned by referencing the input variable media_category_name

### 3. Create a Media Asset

Add the following script in the TF file (such as main.tf) to create a media asset and upload the source file by pulling it from a URL:

```hcl
# Create a media asset resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "media_asset_name" {
  description = "The name of the media asset"
  type        = string
}

variable "media_asset_url" {
  description = "The HTTP or HTTPS URL of the media source file"
  type        = string
}

variable "media_asset_description" {
  description = "The description of the media asset"
  type        = string
  default     = ""
}

variable "media_asset_labels" {
  description = "The labels of the media asset, separated by commas"
  type        = string
  default     = "tf_label_1,tf_label_2"
}

resource "huaweicloud_vod_media_asset" "test" {
  name        = var.media_asset_name
  media_type  = "MP4"
  url         = var.media_asset_url
  description = var.media_asset_description
  category_id = huaweicloud_vod_media_category.test.id
  labels      = var.media_asset_labels
}
```

**Parameter Description**:
- **name**: The name of the media asset, assigned by referencing the input variable media_asset_name
- **media_type**: The media type of the source file, fixed to MP4 here, which must match the actual format of the source file; this parameter cannot be updated and changing it will recreate the media asset
- **url**: The HTTP or HTTPS URL of the media source file, assigned by referencing the input variable media_asset_url; URL pulling is asynchronous, so the creation succeeds even if the URL is temporarily inaccessible
- **description**: The description of the media asset, assigned by referencing the input variable media_asset_description
- **category_id**: The ID of the category to which the media asset belongs, assigned by referencing the id of the resource huaweicloud_vod_media_category.test; if not specified, the media asset is classified into the system preset "Other" category
- **labels**: The labels of the media asset, separated by commas, assigned by referencing the input variable media_asset_labels

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "<YOUR_ACCESS_KEY>"
secret_key  = "<YOUR_SECRET_KEY>"

# Resource variables
media_category_name     = "tf_test_vod_asset_category"
media_asset_name        = "tf_test_vod_media_asset"
media_asset_url         = "https://test-videos.co.uk/vids/bigbuckbunny/mp4/h264/360/Big_Buck_Bunny_360_10s_1MB.mp4"
media_asset_description = "Created by Terraform for VOD best practice example"
media_asset_labels      = "tf_label_1,tf_label_2"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="media_category_name=my-category"`
2. Environment variables: `export TF_VAR_media_category_name=my-category`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the media category and media asset
4. Run `terraform show` to view the created media category and media asset

## Reference Information

- [Huawei Cloud Video on Demand Product Documentation](https://support.huaweicloud.com/vod/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For VOD Media Category and Media Asset](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vod/media-asset-with-category)
