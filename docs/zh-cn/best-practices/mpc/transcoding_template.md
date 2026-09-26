# 部署转码模板

## 应用场景

媒体处理中心（Media Processing Center，MPC）是华为云提供的一站式媒体处理服务，支持音视频转码、截图、水印、视频加密等能力，帮助用户将上传的源音视频文件转换为适配不同终端与网络环境的输出格式。转码模板用于定义单路输出质量的音视频参数，包括输出封装格式、音频编解码与采样率、视频编解码与分辨率等，是构建转码任务的基础配置。

本最佳实践将介绍如何使用Terraform自动化部署一个MPC转码模板，通过输入变量灵活配置输出格式、音频参数与视频参数，从而满足不同清晰度与播放场景下的转码需求。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [转码模板（huaweicloud_mpc_transcoding_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/mpc_transcoding_template)

### 资源/数据源依赖关系

```
huaweicloud_mpc_transcoding_template
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建转码模板

在TF文件（如main.tf）中添加以下脚本以创建转码模板：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建转码模板资源
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

**参数说明**：
- **name**：转码模板名称，通过引用输入变量 template_name 进行赋值
- **low_bitrate_hd**：是否开启低码率高清，通过引用输入变量 low_bitrate_hd 进行赋值
- **dash_segment_duration**：DASH分片时长（单位：秒），通过引用输入变量 dash_segment_duration 进行赋值
- **hls_segment_duration**：HLS分片时长（单位：秒），通过引用输入变量 hls_segment_duration 进行赋值
- **output_format**：输出封装格式，通过引用输入变量 output_format 进行赋值
- **audio.bitrate**：音频码率，通过引用输入变量 audio_bitrate 进行赋值
- **audio.channels**：音频声道数，通过引用输入变量 audio_channels 进行赋值
- **audio.codec**：音频编码格式，通过引用输入变量 audio_codec 进行赋值
- **audio.output_policy**：音频输出策略，通过引用输入变量 audio_output_policy 进行赋值
- **audio.sample_rate**：音频采样率，通过引用输入变量 audio_sample_rate 进行赋值
- **video.max_consecutive_bframes**：连续B帧的最大数量，通过引用输入变量 video_max_consecutive_bframes 进行赋值
- **video.bitrate**：视频码率，通过引用输入变量 video_bitrate 进行赋值
- **video.black_bar_removal**：是否开启黑边去除，通过引用输入变量 video_black_bar_removal 进行赋值
- **video.codec**：视频编码格式，通过引用输入变量 video_codec 进行赋值
- **video.fps**：视频帧率，通过引用输入变量 video_fps 进行赋值
- **video.level**：视频编码级别，通过引用输入变量 video_level 进行赋值
- **video.max_iframes_interval**：I帧的最大间隔，通过引用输入变量 video_max_iframes_interval 进行赋值
- **video.output_policy**：视频输出策略，通过引用输入变量 video_output_policy 进行赋值
- **video.quality**：视频质量，通过引用输入变量 video_quality 进行赋值
- **video.profile**：视频编码档次，通过引用输入变量 video_profile 进行赋值
- **video.height**：视频高度，通过引用输入变量 video_height 进行赋值
- **video.width**：视频宽度，通过引用输入变量 video_width 进行赋值

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
template_name = "your_transcoding_template_name"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="template_name=my-template"`
2. 环境变量：`export TF_VAR_template_name=my-template`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建转码模板
4. 运行 `terraform show` 查看已创建的转码模板

## 参考信息

- [华为云媒体处理中心产品文档](https://support.huaweicloud.com/mpc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [MPC转码模板最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/mpc/transcoding-template)
