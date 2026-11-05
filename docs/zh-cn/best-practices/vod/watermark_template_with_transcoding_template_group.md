# 部署水印模板与转码模板组

## 应用场景

视频点播（Video on Demand，VOD）是华为云提供的一站式视频点播服务，支持音视频上传、转码、截图、水印、加密、加速分发与播放鉴权等能力。在在线教育、企业培训、电商视频等场景中，用户通常希望在转码输出的视频上叠加统一的品牌水印，以保证内容版权与品牌一致性。

本最佳实践将介绍如何使用Terraform创建VOD水印模板，并创建引用该水印模板的转码模板组，实现转码输出视频自动叠加水印。水印模板与转码模板组均为配置类资源，创建过程不会产生费用。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [水印模板（huaweicloud_vod_watermark_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_watermark_template)
- [转码模板组（huaweicloud_vod_transcoding_template_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_transcoding_template_group)

### 资源/数据源依赖关系

```
huaweicloud_vod_watermark_template
    └── huaweicloud_vod_transcoding_template_group
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建水印模板

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建水印模板资源
variable "watermark_template_name" {
  description = "水印模板的名称"
  type        = string
}

variable "watermark_template_image_file" {
  description = "本地PNG格式水印图片文件的绝对路径"
  type        = string
}

resource "huaweicloud_vod_watermark_template" "test" {
  name       = var.watermark_template_name
  image_file = var.watermark_template_image_file
  image_type = "PNG"
}
```

**参数说明**：
- **name**：水印模板名称，通过引用输入变量 watermark_template_name 进行赋值
- **image_file**：本地水印图片文件的绝对路径，通过引用输入变量 watermark_template_image_file 进行赋值，Provider会读取该文件并上传至VOD接口返回的上传地址，文件不存在时创建将失败
- **image_type**：水印图片类型，此处固定为PNG

> 注意：水印模板的 image_file 与 image_type 参数不支持更新，修改后会重建模板；水印模板创建包含创建模板、上传图片、确认上传三个步骤，若图片上传失败，控制台可能残留不完整的模板，需要手工删除。

### 3. 创建转码模板组

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建转码模板组资源
variable "template_group_name" {
  description = "转码模板组的名称"
  type        = string
}

variable "template_group_description" {
  description = "转码模板组的描述"
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

**参数说明**：
- **name**：转码模板组名称，通过引用输入变量 template_group_name 进行赋值
- **description**：转码模板组描述，通过引用输入变量 template_group_description 进行赋值
- **audio_codec**：音频编码格式，此处为HEAAC1
- **video_codec**：视频编码格式，此处为H265
- **watermark_template_ids**：引用的水印模板ID列表，通过引用 huaweicloud_vod_watermark_template.test.id 进行赋值，转码输出视频将叠加该水印
- **quality_info.output_format**：输出格式，可选值为MP4、DASH、DASH_HLS、MP3、ADTS和HLS，此处为MP4
- **quality_info.audio.channels**：音频声道数，此处为1
- **quality_info.audio.sample_rate**：音频采样率，此处为2
- **quality_info.video.bitrate**：视频码率，此处为1000
- **quality_info.video.frame_rate**：视频帧率，此处为1
- **quality_info.video.height**：视频高度，此处为1080
- **quality_info.video.width**：视频宽度，此处为1920
- **quality_info.video.quality**：视频质量，可选值为4K、2K、FHD、HD、SD和LD，此处为FHD

> 注意：转码模板组通过 watermark_template_ids 引用水印模板，因此执行 destroy 时会自动先删除转码模板组。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
watermark_template_name       = "tf_test_vod_watermark_template"
watermark_template_image_file = "D:/styles/watermark.png"
template_group_name           = "tf_test_vod_watermark_group"
template_group_description    = "Created by Terraform for VOD watermark example"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="watermark_template_name=my-watermark"`
2. 环境变量：`export TF_VAR_watermark_template_name=my-watermark`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建水印模板与转码模板组
4. 运行 `terraform show` 查看已创建的水印模板与转码模板组

## 参考信息

- [华为云视频点播产品文档](https://support.huaweicloud.com/vod/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VOD水印模板与转码模板组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vod/watermark-template-with-transcoding-template-group)
