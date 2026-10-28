# Deploy Watermark Template and Transcoding Template Group

## Application Scenario

Video on Demand (VOD) is a one-stop video on-demand service provided by Huawei Cloud, supporting audio and video upload, transcoding, screenshot, watermark, encryption, CDN acceleration and distribution, and playback authentication. In scenarios such as online education, enterprise training, and e-commerce video, users usually want to overlay a unified brand watermark on transcoded output videos to ensure content copyright and brand consistency.

This best practice will introduce how to use Terraform to create a VOD watermark template and create a transcoding template group that references the watermark template, so that transcoded output videos automatically carry the watermark. Both the watermark template and the transcoding template group are configuration-type resources, and creating them does not generate any fee.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Watermark Template (huaweicloud_vod_watermark_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_watermark_template)
- [Transcoding Template Group (huaweicloud_vod_transcoding_template_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_transcoding_template_group)

### Resource/Data Source Dependencies

```
huaweicloud_vod_watermark_template
    └── huaweicloud_vod_transcoding_template_group
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Watermark Template

Add the following script to the TF file (such as main.tf):

```hcl
# Create a watermark template resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "watermark_template_name" {
  description = "The name of the watermark template"
  type        = string
}

variable "watermark_template_image_file" {
  description = "The absolute path of the local watermark image file in PNG format"
  type        = string
}

resource "huaweicloud_vod_watermark_template" "test" {
  name       = var.watermark_template_name
  image_file = var.watermark_template_image_file
  image_type = "PNG"
}
```

**Parameter Description**:
- **name**: The name of the watermark template, assigned by referencing the input variable watermark_template_name
- **image_file**: The absolute path of the local watermark image file, assigned by referencing the input variable watermark_template_image_file. The provider reads this file and uploads it to the upload URL returned by the VOD API; the creation fails if the file does not exist
- **image_type**: The watermark image type, fixed to PNG here

> Note: The image_file and image_type arguments of the watermark template cannot be updated, and changing them will recreate the template. The creation of the watermark template includes three steps (creating the template, uploading the image and confirming the upload); if the image upload fails, an incomplete template may remain in the console and needs to be deleted manually.

### 3. Create a Transcoding Template Group

Add the following script to the TF file (such as main.tf):

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
  name                   = var.template_group_name
  description            = var.template_group_description
  audio_codec            = "HEAAC1"
  video_codec            = "H265"
  watermark_template_ids = [huaweicloud_vod_watermark_template.test.id]

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
- **name**: The name of the transcoding template group, assigned by referencing the input variable template_group_name
- **description**: The description of the transcoding template group, assigned by referencing the input variable template_group_description
- **audio_codec**: The audio codec, set to HEAAC1 here
- **video_codec**: The video codec, set to H265 here
- **watermark_template_ids**: The list of referenced watermark template IDs, assigned by referencing huaweicloud_vod_watermark_template.test.id, so that transcoded output videos carry the watermark
- **quality_info.output_format**: The output format, valid values are MP4, DASH, DASH_HLS, MP3, ADTS and HLS, set to MP4 here
- **quality_info.audio.channels**: The number of audio channels, set to 1 here
- **quality_info.audio.sample_rate**: The audio sample rate, set to 2 here
- **quality_info.video.bitrate**: The video bitrate, set to 1000 here
- **quality_info.video.frame_rate**: The video frame rate, set to 1 here
- **quality_info.video.height**: The video height, set to 1080 here
- **quality_info.video.width**: The video width, set to 1920 here
- **quality_info.video.quality**: The video quality, valid values are 4K, 2K, FHD, HD, SD and LD, set to FHD here

> Note: The transcoding template group references the watermark template through watermark_template_ids, so destroy will delete the template group first automatically.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
watermark_template_name       = "tf_test_vod_watermark_template"
watermark_template_image_file = "D:/styles/watermark.png"
template_group_name           = "tf_test_vod_watermark_group"
template_group_description    = "Created by Terraform for VOD watermark example"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="watermark_template_name=my-watermark"`
2. Environment variables: `export TF_VAR_watermark_template_name=my-watermark`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the watermark template and transcoding template group
4. Run `terraform show` to view the created watermark template and transcoding template group

## Reference Information

- [Huawei Cloud Video on Demand Product Documentation](https://support.huaweicloud.com/vod/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For VOD Watermark Template and Transcoding Template Group](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vod/watermark-template-with-transcoding-template-group)
