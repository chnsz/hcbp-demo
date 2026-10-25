# Deploy Media Category and Transcoding Template Group

## Application Scenario

Video on Demand (VOD) is a one-stop video on-demand service provided by Huawei Cloud for scenarios such as audio and video websites, online education, e-commerce live streaming, and enterprise training. It delivers end-to-end capabilities from audio and video upload, transcoding, and storage to acceleration, distribution, and playback. In real-world business, users usually need to group massive audio and video resources and predefine transcoding template groups so that subsequently uploaded source files can be transcoded and distributed with unified output specifications.

This best practice will introduce how to use Terraform to automatically deploy a VOD media category and a transcoding template group, including media category creation and transcoding template group creation with audio and video quality information. Both the media category and the transcoding template group are configuration-type resources, and creating them does not generate any fee.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Media Category (huaweicloud_vod_media_category)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_media_category)
- [Transcoding Template Group (huaweicloud_vod_transcoding_template_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_transcoding_template_group)

### Resource/Data Source Dependencies

```
huaweicloud_vod_media_category

huaweicloud_vod_transcoding_template_group
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create a Media Category

Add the following script in the TF file (such as main.tf):

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
- **name**: The name of the media category, assigned by referencing the input variable media_category_name.

> Note: The `parent_id` argument of the media category cannot be updated, and changing it will recreate the category. The parent category ID of a top-level category is `0` by default, and a media category that still contains child categories or media assets cannot be deleted.

### 3. Create a Transcoding Template Group

Add the following script in the TF file (such as main.tf):

```hcl
# Create a transcoding template group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "template_group_name" {
  description = "The name of the transcoding template group"
  type        = string
}

variable "template_group_description" {
  description = "The description of the transcoding template group"
  type        = string
  default     = ""
}

resource "huaweicloud_vod_transcoding_template_group" "test" {
  name        = var.template_group_name
  description = var.template_group_description
  audio_codec = "HEAAC1"
  video_codec = "H265"

  quality_info {
    output_format = "MP4"

    audio {
      channels    = 1
      sample_rate = 2
    }

    video {
      bitrate    = 1000
      frame_rate = 1
      height     = 1080
      quality    = "FHD"
      width      = 1920
    }
  }
}
```

**Parameter Description**:
- **name**: The name of the transcoding template group, assigned by referencing the input variable template_group_name.
- **description**: The description of the transcoding template group, assigned by referencing the input variable template_group_description.
- **audio_codec**: The audio codec, set to `HEAAC1` here.
- **video_codec**: The video codec, set to `H265` here.
- **quality_info.output_format**: The output format, set to `MP4` here. Valid values include `MP4`, `DASH`, `DASH_HLS`, `MP3`, `ADTS`, and `HLS`.
- **quality_info.audio.channels**: The number of audio channels, set to `1` here.
- **quality_info.audio.sample_rate**: The audio sample rate, an enumerated number where `1` means `AUTO`, `2` means `22,050` Hz, `3` means `32,000` Hz, `4` means `44,100` Hz, `5` means `48,000` Hz, and `6` means `96,000` Hz; set to `2` here.
- **quality_info.video.bitrate**: The video bitrate, set to `1000` here.
- **quality_info.video.frame_rate**: The video frame rate, set to `1` here.
- **quality_info.video.height**: The video height, set to `1080` here.
- **quality_info.video.width**: The video width, set to `1920` here.
- **quality_info.video.quality**: The video quality, set to `FHD` here. Valid values include `4K`, `2K`, `FHD`, `HD`, `SD`, and `LD`.

> Note: The `quality_info` block is required. If the output format is `HLS`, the `hls_segment_duration` argument can be set to specify the segment duration, and valid values are `2`, `3`, `5`, and `10`. The `watermark_template_ids` argument requires existing watermark templates, so this practice does not use it.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
media_category_name        = "tf_test_vod_media_category"
template_group_name        = "tf_test_vod_template_group"
template_group_description = "Created by Terraform for VOD best practice example"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the media category and transcoding template group
4. Run `terraform show` to view the created media category and transcoding template group

## Reference Information

- [Huawei Cloud Video on Demand Product Documentation](https://support.huaweicloud.com/vod/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For VOD Media Category and Transcoding Template Group](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vod/media-category-with-transcoding-template-group)
