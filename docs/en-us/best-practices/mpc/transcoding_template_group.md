# Deploy Transcoding Template Group

## Application Scenario

Media Processing Center (MPC) is a one-stop media processing service provided by Huawei Cloud for the production, processing, and distribution of audio and video content. A transcoding template group packages a set of common audio parameters, shared video parameters, and multiple output qualities into a reusable configuration. Transcoding tasks can reference the template group directly to produce multi-bitrate and multi-resolution outputs, avoiding repeated parameter configuration for each output specification.

This best practice will introduce how to use Terraform to automatically deploy an MPC transcoding template group, including output format and segment duration configuration, audio parameter configuration, shared video parameter configuration, and multiple video output quality definitions.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Transcoding Template Group (huaweicloud_mpc_transcoding_template_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/mpc_transcoding_template_group)

### Resource/Data Source Dependencies

```
huaweicloud_mpc_transcoding_template_group
```

## Operation Steps

### 1. Script Preparation

Prepare a TF file (such as main.tf) in the specified workspace for writing the scripts of the current best practice, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Transcoding Template Group

Add the following script to the TF file (such as main.tf):

```hcl
# Create a transcoding template group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "template_name" {
  description = "The name of the transcoding template group"
  type        = string
}

variable "low_bitrate_hd" {
  description = "Whether to enable low bitrate HD. true: enable, false: disable"
  type        = bool
  default     = true
}

variable "dash_segment_duration" {
  description = "The DASH segment duration in seconds"
  type        = number
  default     = 5
}

variable "hls_segment_duration" {
  description = "The HLS segment duration in seconds"
  type        = number
  default     = 5
}

variable "output_format" {
  description = "The output format. 1: HLS, 2: DASH, 3: HLS+DASH, 4: MP4, 5: MP3, 6: ADTS"
  type        = number
  default     = 1
}

variable "audio_bitrate" {
  description = "The audio bitrate. 0: auto"
  type        = number
  default     = 0
}

variable "audio_channels" {
  description = "The audio channels. 1: AUTO, 2: mono, 6: stereo"
  type        = number
  default     = 2
}

variable "audio_codec" {
  description = "The audio codec. 1: AAC, 2: HEAAC1, 3: HEAAC2, 4: MP3"
  type        = number
  default     = 2
}

variable "audio_output_policy" {
  description = "The audio output policy. transcode: transcoding, copy: passthrough, discard: discard"
  type        = string
  default     = "transcode"
}

variable "audio_sample_rate" {
  description = "The audio sample rate. 1: AUTO, 2: 22050Hz, 3: 32000Hz, 4: 44100Hz, 5: 48000Hz, 6: 96000Hz"
  type        = number
  default     = 1
}

variable "video_max_consecutive_bframes" {
  description = "The maximum number of consecutive B-frames"
  type        = number
  default     = 7
}

variable "video_black_bar_removal" {
  description = "Whether to remove black bars. 0: disable, 1: enable"
  type        = number
  default     = 0
}

variable "video_codec" {
  description = "The video codec. 1: H.264, 2: H.265"
  type        = number
  default     = 2
}

variable "video_fps" {
  description = "The video frame rate. 0: auto"
  type        = number
  default     = 0
}

variable "video_level" {
  description = "The video level. 15: default"
  type        = number
  default     = 15
}

variable "video_max_iframes_interval" {
  description = "The maximum interval between I-frames"
  type        = number
  default     = 5
}

variable "video_output_policy" {
  description = "The video output policy. transcode: transcoding, copy: passthrough, discard: discard"
  type        = string
  default     = "transcode"
}

variable "video_quality" {
  description = "The video quality. 1: VBR, 2: CBR"
  type        = number
  default     = 1
}

variable "video_profile" {
  description = "The video profile. 1: baseline, 2: main, 3: high, 4: default"
  type        = number
  default     = 4
}

variable "video_outputs" {
  description = "The list of video output definitions"
  type        = list(object({
    width   = number
    height  = number
    bitrate = number
  }))
  default     = [
    {
      width   = 1920
      height  = 1080
      bitrate = 0
    }
  ]
  nullable    = false
}

resource "huaweicloud_mpc_transcoding_template_group" "test" {
  name                  = var.template_name
  low_bitrate_hd        = var.low_bitrate_hd
  dash_segment_duration = var.dash_segment_duration
  hls_segment_duration  = var.hls_segment_duration
  output_format         = var.output_format

  audio {
    bitrate       = var.audio_bitrate
    channels      = var.audio_channels
    codec         = var.audio_codec
    output_policy = var.audio_output_policy
    sample_rate   = var.audio_sample_rate
  }

  video_common {
    max_consecutive_bframes = var.video_max_consecutive_bframes
    black_bar_removal       = var.video_black_bar_removal
    codec                   = var.video_codec
    fps                     = var.video_fps
    level                   = var.video_level
    max_iframes_interval    = var.video_max_iframes_interval
    output_policy           = var.video_output_policy
    quality                 = var.video_quality
    profile                 = var.video_profile
  }

  dynamic "videos" {
    for_each = var.video_outputs

    content {
      width   = videos.value.width
      height  = videos.value.height
      bitrate = videos.value.bitrate
    }
  }
}
```

**Parameter Description**:

- **name**: The name of the transcoding template group, assigned by referencing the input variable template_name
- **low_bitrate_hd**: Whether to enable low bitrate HD, assigned by referencing the input variable low_bitrate_hd
- **dash_segment_duration**: The DASH segment duration, assigned by referencing the input variable dash_segment_duration
- **hls_segment_duration**: The HLS segment duration, assigned by referencing the input variable hls_segment_duration
- **output_format**: The output format, assigned by referencing the input variable output_format
- **audio**: The audio parameter configuration block
  - **bitrate**: The audio bitrate, assigned by referencing the input variable audio_bitrate
  - **channels**: The audio channels, assigned by referencing the input variable audio_channels
  - **codec**: The audio codec, assigned by referencing the input variable audio_codec
  - **output_policy**: The audio output policy, assigned by referencing the input variable audio_output_policy
  - **sample_rate**: The audio sample rate, assigned by referencing the input variable audio_sample_rate
- **video_common**: The shared video parameter configuration block
  - **max_consecutive_bframes**: The maximum number of consecutive B-frames, assigned by referencing the input variable video_max_consecutive_bframes
  - **black_bar_removal**: Whether to remove black bars, assigned by referencing the input variable video_black_bar_removal
  - **codec**: The video codec, assigned by referencing the input variable video_codec
  - **fps**: The video frame rate, assigned by referencing the input variable video_fps
  - **level**: The video level, assigned by referencing the input variable video_level
  - **max_iframes_interval**: The maximum interval between I-frames, assigned by referencing the input variable video_max_iframes_interval
  - **output_policy**: The video output policy, assigned by referencing the input variable video_output_policy
  - **quality**: The video quality, assigned by referencing the input variable video_quality
  - **profile**: The video profile, assigned by referencing the input variable video_profile
- **videos**: The video output definitions, assigned by iterating the input variable video_outputs through a dynamic block, where each element contains width, height, and bitrate

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
template_name = "your_transcoding_template_group_name"
video_outputs = [
  {
    width   = 1920
    height  = 1080
    bitrate = 0
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names require adding a `.auto` prefix, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="template_name=my-template-group"`
2. Environment variables: `export TF_VAR_template_name=my-template-group`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the transcoding template group
4. Run `terraform show` to view the created transcoding template group

## Reference Information

- [Huawei Cloud Media Processing Center Product Documentation](https://support.huaweicloud.com/mpc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For MPC Transcoding Template Group](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/mpc/transcoding-template-group)
