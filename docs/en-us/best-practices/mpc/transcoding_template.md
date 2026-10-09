# Deploy Transcoding Template

## Application Scenario

Media Processing Center (MPC) is a one-stop media processing service provided by Huawei Cloud, supporting audio and video transcoding, screenshot, watermarking, and video encryption. It helps users convert uploaded source audio and video files into output formats suitable for different terminals and network conditions. A transcoding template defines the audio and video parameters for a single output quality, including the output container format, audio codec and sample rate, and video codec and resolution, and serves as the basic configuration for building transcoding tasks.

This best practice will introduce how to use Terraform to automatically deploy an MPC transcoding template, flexibly configuring the output format, audio parameters, and video parameters through input variables, so as to meet transcoding requirements for different quality levels and playback scenarios.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Transcoding Template (huaweicloud_mpc_transcoding_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/mpc_transcoding_template)

### Resource/Data Source Dependencies

```
huaweicloud_mpc_transcoding_template
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Transcoding Template

Add the following script to the TF file (such as main.tf) to create a transcoding template:

```hcl
# Create a transcoding template resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "template_name" {
  description = "The name of the transcoding template"
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

variable "video_bitrate" {
  description = "The video bitrate. 0: auto"
  type        = number
  default     = 0
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

variable "video_height" {
  description = "The video height. 0: auto"
  type        = number
  default     = 0
}

variable "video_width" {
  description = "The video width. 0: auto"
  type        = number
  default     = 0
}

resource "huaweicloud_mpc_transcoding_template" "test" {
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

  video {
    max_consecutive_bframes = var.video_max_consecutive_bframes
    bitrate                 = var.video_bitrate
    black_bar_removal       = var.video_black_bar_removal
    codec                   = var.video_codec
    fps                     = var.video_fps
    level                   = var.video_level
    max_iframes_interval    = var.video_max_iframes_interval
    output_policy           = var.video_output_policy
    quality                 = var.video_quality
    profile                 = var.video_profile
    height                  = var.video_height
    width                   = var.video_width
  }
}
```

**Parameter Description**:
- **name**: The name of the transcoding template, assigned by referencing the input variable template_name
- **low_bitrate_hd**: Whether to enable low bitrate HD, assigned by referencing the input variable low_bitrate_hd
- **dash_segment_duration**: The DASH segment duration in seconds, assigned by referencing the input variable dash_segment_duration
- **hls_segment_duration**: The HLS segment duration in seconds, assigned by referencing the input variable hls_segment_duration
- **output_format**: The output format, assigned by referencing the input variable output_format
- **audio.bitrate**: The audio bitrate, assigned by referencing the input variable audio_bitrate
- **audio.channels**: The audio channels, assigned by referencing the input variable audio_channels
- **audio.codec**: The audio codec, assigned by referencing the input variable audio_codec
- **audio.output_policy**: The audio output policy, assigned by referencing the input variable audio_output_policy
- **audio.sample_rate**: The audio sample rate, assigned by referencing the input variable audio_sample_rate
- **video.max_consecutive_bframes**: The maximum number of consecutive B-frames, assigned by referencing the input variable video_max_consecutive_bframes
- **video.bitrate**: The video bitrate, assigned by referencing the input variable video_bitrate
- **video.black_bar_removal**: Whether to remove black bars, assigned by referencing the input variable video_black_bar_removal
- **video.codec**: The video codec, assigned by referencing the input variable video_codec
- **video.fps**: The video frame rate, assigned by referencing the input variable video_fps
- **video.level**: The video level, assigned by referencing the input variable video_level
- **video.max_iframes_interval**: The maximum interval between I-frames, assigned by referencing the input variable video_max_iframes_interval
- **video.output_policy**: The video output policy, assigned by referencing the input variable video_output_policy
- **video.quality**: The video quality, assigned by referencing the input variable video_quality
- **video.profile**: The video profile, assigned by referencing the input variable video_profile
- **video.height**: The video height, assigned by referencing the input variable video_height
- **video.width**: The video width, assigned by referencing the input variable video_width

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
template_name = "your_transcoding_template_name"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="template_name=my-template"`
2. Environment variables: `export TF_VAR_template_name=my-template`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the transcoding template
4. Run `terraform show` to view the created transcoding template

## Reference Information

- [Huawei Cloud Media Processing Center Product Documentation](https://support.huaweicloud.com/mpc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For MPC Transcoding Template](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/mpc/transcoding-template)
