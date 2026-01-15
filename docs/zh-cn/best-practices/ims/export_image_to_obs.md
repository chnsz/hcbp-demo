# 部署导出镜像到OBS

## 应用场景

镜像服务（Image Management Service，IMS）是华为云提供的镜像管理服务，支持镜像的创建、共享、复制、导出等功能。通过导出镜像到OBS，可以将私有镜像导出为镜像文件并存储到OBS桶中，实现镜像的备份、迁移和离线存储。本最佳实践将介绍如何使用Terraform自动化部署导出镜像到OBS，包括查询私有镜像、创建OBS桶和导出镜像到OBS桶。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [镜像数据源（huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [OBS桶资源（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [镜像导出资源（huaweicloud_ims_image_export）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ims_image_export)

### 资源/数据源依赖关系

```text
data.huaweicloud_images_images
    └── huaweicloud_obs_bucket
        └── huaweicloud_ims_image_export
```

> 注意：镜像导出需要先创建OBS桶。导出的镜像文件会存储在指定的OBS桶中，支持多种文件格式，包括vhd、zvhd、vmdk、raw、qcow2、zvhd2、vdi等。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询私有镜像

在TF文件（如main.tf）中添加以下脚本以查询私有镜像：

```hcl
variable "region_name" {
  description = "The region where the CDN domain is located"
  type        = string
}

variable "image_type" {
  description = "The type of images to filter (ECS, BMS, etc.)"
  type        = string
  default     = ""
}

variable "image_os" {
  description = "The OS of images to filter (Ubuntu, CentOS, etc.)"
  type        = string
  default     = ""
}

variable "image_name_regex" {
  description = "The regex pattern to filter images by name"
  type        = string
  default     = ""
}

# 查询指定region下当前用户拥有的所有私有镜像
data "huaweicloud_images_images" "test" {
  visibility = "private"
  image_type = var.image_type != "" ? var.image_type : null
  os         = var.image_os != "" ? var.image_os : null
  name_regex = var.image_name_regex != "" ? var.image_name_regex : null
}

# 过滤出状态为active的镜像
locals {
  active_images = [
    for image in data.huaweicloud_images_images.test.images : image if image.status == "active"
  ]
}
```

**参数说明**：
- **visibility**：镜像可见性，设置为"private"表示查询私有镜像
- **image_type**：镜像类型，通过引用输入变量image_type进行赋值，可选参数，用于过滤特定类型的镜像（如ECS、BMS等）
- **os**：操作系统类型，通过引用输入变量image_os进行赋值，可选参数，用于过滤特定操作系统的镜像（如Ubuntu、CentOS等）
- **name_regex**：镜像名称正则表达式，通过引用输入变量image_name_regex进行赋值，可选参数，用于按名称过滤镜像
- **active_images**：本地变量，用于过滤出状态为"active"的镜像，只有状态为active的镜像才能导出

> 注意：只有状态为"active"的镜像才能导出。可以通过设置image_type、image_os、name_regex等参数来过滤需要导出的镜像。

### 3. 创建OBS桶

在TF文件（如main.tf）中添加以下脚本以创建OBS桶：

```hcl
variable "obs_bucket_name" {
  description = "The name of the OBS bucket for storing exported images"
  type        = string
}

variable "obs_bucket_tags" {
  description = "The tags of the OBS bucket"
  type        = map(string)
  default     = {}
}

# 在指定region下创建OBS桶资源，用于存储导出的镜像文件
resource "huaweicloud_obs_bucket" "test" {
  bucket        = var.obs_bucket_name
  storage_class = "STANDARD"
  region        = var.region_name

  tags = var.obs_bucket_tags
}
```

**参数说明**：
- **bucket**：OBS桶名称，通过引用输入变量obs_bucket_name进行赋值，全局唯一
- **storage_class**：存储类型，设置为"STANDARD"表示标准存储
- **region**：区域，通过引用输入变量region_name进行赋值
- **tags**：标签，通过引用输入变量obs_bucket_tags进行赋值，可选参数

> 注意：OBS桶名称需要全局唯一，建议使用有意义的命名规则。存储类型可以根据实际需求选择，标准存储适合频繁访问的场景。

### 4. 导出镜像到OBS

在TF文件（如main.tf）中添加以下脚本以导出镜像到OBS：

```hcl
variable "file_format" {
  description = "The file format of the exported image (vhd, zvhd, vmdk, raw, qcow2, zvhd2, vdi)"
  type        = string
  default     = "zvhd2"
}

# 在指定region下创建镜像导出资源，将每个镜像导出到OBS桶
resource "huaweicloud_ims_image_export" "test" {
  count = length(local.active_images)

  region      = var.region_name
  image_id    = local.active_images[count.index].id
  bucket_url  = "${huaweicloud_obs_bucket.test.bucket}:${local.active_images[count.index].name}-${local.active_images[count.index].id}.${var.file_format}"
  file_format = var.file_format

  depends_on = [huaweicloud_obs_bucket.test]
}
```

**参数说明**：
- **count**：资源创建数量，通过引用本地变量active_images的长度进行赋值，为每个active镜像创建一个导出任务
- **region**：区域，通过引用输入变量region_name进行赋值
- **image_id**：镜像ID，通过引用本地变量active_images进行赋值
- **bucket_url**：OBS桶URL，格式为"bucket:object_key"，通过组合OBS桶名称、镜像名称、镜像ID和文件格式生成
- **file_format**：文件格式，通过引用输入变量file_format进行赋值，支持vhd、zvhd、vmdk、raw、qcow2、zvhd2、vdi等格式，默认值为"zvhd2"
- **depends_on**：显式依赖关系，确保在OBS桶创建后再导出镜像

> 注意：镜像导出需要较长时间，请耐心等待。导出的镜像文件会存储在指定的OBS桶中，文件名格式为"镜像名称-镜像ID.文件格式"。支持的文件格式包括vhd、zvhd、vmdk、raw、qcow2、zvhd2、vdi等，建议根据实际需求选择合适的格式。

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# OBS桶配置
obs_bucket_name = "my-image-backup-bucket-test"

# 镜像导出配置
file_format = "zvhd2"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `region_name`需要设置为资源所在的区域
   - `obs_bucket_name`需要设置为OBS桶名称，需要全局唯一
   - `file_format`可以设置导出的文件格式，支持vhd、zvhd、vmdk、raw、qcow2、zvhd2、vdi等格式，默认值为"zvhd2"
   - `image_type`可以设置镜像类型过滤条件，可选参数
   - `image_os`可以设置操作系统类型过滤条件，可选参数
   - `image_name_regex`可以设置镜像名称正则表达式过滤条件，可选参数
   - `obs_bucket_tags`可以设置OBS桶标签，可选参数
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="obs_bucket_name=my-bucket" -var="file_format=zvhd2"`
2. 环境变量：`export TF_VAR_obs_bucket_name=my-bucket` 和 `export TF_VAR_file_format=zvhd2`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。镜像导出需要较长时间，请耐心等待。导出的镜像文件会存储在指定的OBS桶中，可以通过OBS控制台或API下载使用。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建导出镜像到OBS：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建OBS桶和导出镜像
4. 运行 `terraform show` 查看已创建的导出镜像到OBS详情

> 注意：镜像导出需要较长时间，请耐心等待。导出的镜像文件会存储在指定的OBS桶中，可以通过OBS控制台或API下载使用。只有状态为"active"的镜像才能导出，请确保镜像状态正确。

## 参考信息

- [华为云IMS产品文档](https://support.huaweicloud.com/ims/index.html)
- [华为云OBS产品文档](https://support.huaweicloud.com/obs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [导出镜像到OBS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ims/export-image-to-obs)
