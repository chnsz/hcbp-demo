# 部署媒资分类与转码模板组

## 应用场景

视频点播（Video on Demand，VOD）是华为云提供的一站式视频点播服务，面向音视频网站、在线教育、电商直播、企业培训等场景，提供从音视频上传、转码、存储、加速分发到播放的端到端能力。在实际业务中，用户通常需要先对海量音视频资源进行分组管理，并预先定义好转码模板组，以便后续上传的源文件能够按照统一的输出规格进行转码与分发。

本最佳实践将介绍如何使用Terraform自动化部署VOD媒资分类与转码模板组，包括媒资分类创建以及转码模板组创建（含音频、视频质量信息配置）。媒资分类与转码模板组均为配置类资源，创建时不会产生费用。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [媒资分类（huaweicloud_vod_media_category）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_media_category)
- [转码模板组（huaweicloud_vod_transcoding_template_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_transcoding_template_group)

### 资源/数据源依赖关系

```
huaweicloud_vod_media_category

huaweicloud_vod_transcoding_template_group
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建媒资分类

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建媒资分类资源
variable "media_category_name" {
  description = "The name of the media category"
  type        = string
}

resource "huaweicloud_vod_media_category" "test" {
  name = var.media_category_name
}
```

**参数说明**：
- **name**：媒资分类名称，通过引用输入变量 media_category_name 进行赋值。

> 说明：媒资分类的 `parent_id` 参数不支持更新，修改该参数将重建分类；顶级分类的父分类ID默认为 `0`；仍包含子分类或媒资的分类无法删除。

### 3. 创建转码模板组

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建转码模板组资源
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

**参数说明**：
- **name**：转码模板组名称，通过引用输入变量 template_group_name 进行赋值。
- **description**：转码模板组描述，通过引用输入变量 template_group_description 进行赋值。
- **audio_codec**：音频编码格式，此处设置为 `HEAAC1`。
- **video_codec**：视频编码格式，此处设置为 `H265`。
- **quality_info.output_format**：输出格式，此处设置为 `MP4`；有效值包括 `MP4`、`DASH`、`DASH_HLS`、`MP3`、`ADTS` 和 `HLS`。
- **quality_info.audio.channels**：音频声道数，此处设置为 `1`。
- **quality_info.audio.sample_rate**：音频采样率，为枚举数字，`1` 表示 `AUTO`，`2` 表示 `22,050` Hz，`3` 表示 `32,000` Hz，`4` 表示 `44,100` Hz，`5` 表示 `48,000` Hz，`6` 表示 `96,000` Hz；此处设置为 `2`。
- **quality_info.video.bitrate**：视频码率，此处设置为 `1000`。
- **quality_info.video.frame_rate**：视频帧率，此处设置为 `1`。
- **quality_info.video.height**：视频高度，此处设置为 `1080`。
- **quality_info.video.width**：视频宽度，此处设置为 `1920`。
- **quality_info.video.quality**：视频质量，此处设置为 `FHD`；有效值包括 `4K`、`2K`、`FHD`、`HD`、`SD` 和 `LD`。

> 说明：`quality_info` 块为必填项；若输出格式为 `HLS`，可通过 `hls_segment_duration` 参数指定分片时长，有效值为 `2`、`3`、`5` 和 `10`；`watermark_template_ids` 参数需要已存在的水印模板，本实践未使用。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
media_category_name        = "tf_test_vod_media_category"
template_group_name        = "tf_test_vod_template_group"
template_group_description = "Created by Terraform for VOD best practice example"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="media_category_name=my-category"`
2. 环境变量：`export TF_VAR_media_category_name=my-category`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建媒资分类与转码模板组
4. 运行 `terraform show` 查看已创建的媒资分类与转码模板组

## 参考信息

- [华为云视频点播产品文档](https://support.huaweicloud.com/vod/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VOD媒资分类与转码模板组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vod/media-category-with-transcoding-template-group)
