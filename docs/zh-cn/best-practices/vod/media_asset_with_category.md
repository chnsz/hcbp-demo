# 部署媒资分类与媒资

## 应用场景

视频点播（Video on Demand，VOD）是华为云提供的一站式视频点播服务，支持音视频上传、转码、存储、加速分发与播放等能力，帮助企业和开发者快速构建视频点播业务。媒资分类用于对海量音视频资源进行分组管理，媒资则是点播服务中承载音视频源文件的核心对象。

本最佳实践将介绍如何使用Terraform自动化部署VOD媒资分类与媒资，包括创建媒资分类，以及通过URL拉取方式创建媒资并关联到该分类。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [媒资分类（huaweicloud_vod_media_category）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_media_category)
- [媒资（huaweicloud_vod_media_asset）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vod_media_asset)

### 资源/数据源依赖关系

```
huaweicloud_vod_media_category
    └── huaweicloud_vod_media_asset
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建媒资分类

在TF文件（如main.tf）中添加以下脚本以创建媒资分类：

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
- **name**：媒资分类名称，通过引用输入变量 media_category_name 进行赋值

### 3. 创建媒资

在TF文件（如main.tf）中添加以下脚本以创建媒资，并通过URL拉取方式上传源文件：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建媒资资源
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

**参数说明**：
- **name**：媒资名称，通过引用输入变量 media_asset_name 进行赋值
- **media_type**：媒资源文件的媒体类型，此处固定为 MP4，须与源文件实际格式一致，该参数不支持更新，修改后会重建媒资
- **url**：媒资源文件的HTTP或HTTPS地址，通过引用输入变量 media_asset_url 进行赋值，URL拉取为异步方式，即使URL暂时不可访问创建也会成功
- **description**：媒资描述，通过引用输入变量 media_asset_description 进行赋值
- **category_id**：媒资所属分类ID，通过引用资源 huaweicloud_vod_media_category.test 的 id 进行赋值，未指定时媒资将归入系统预置的「其他」分类
- **labels**：媒资标签，多个标签以逗号分隔，通过引用输入变量 media_asset_labels 进行赋值

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "<YOUR_ACCESS_KEY>"
secret_key  = "<YOUR_SECRET_KEY>"

# 资源变量
media_category_name     = "tf_test_vod_asset_category"
media_asset_name        = "tf_test_vod_media_asset"
media_asset_url         = "https://test-videos.co.uk/vids/bigbuckbunny/mp4/h264/360/Big_Buck_Bunny_360_10s_1MB.mp4"
media_asset_description = "Created by Terraform for VOD best practice example"
media_asset_labels      = "tf_label_1,tf_label_2"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建媒资分类与媒资
4. 运行 `terraform show` 查看已创建的媒资分类与媒资

## 参考信息

- [华为云视频点播产品文档](https://support.huaweicloud.com/vod/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VOD媒资分类与媒资最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vod/media-asset-with-category)
