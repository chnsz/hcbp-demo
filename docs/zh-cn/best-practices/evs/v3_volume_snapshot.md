# 部署V3云硬盘与快照

## 应用场景

云硬盘（Elastic Volume Service，EVS）是华为云提供的高性能、高可靠、可扩展的块存储服务，为ECS实例提供持久化存储。快照是云硬盘数据在某一时间点的完整副本，可用于数据备份与快速恢复，是保障业务数据可靠性的重要手段。

本最佳实践将介绍如何使用Terraform自动化部署EVS V3云硬盘与快照，包括可用区与镜像的自动查询、云硬盘的创建以及基于该云硬盘创建快照。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [镜像列表（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [V3云硬盘（huaweicloud_evsv3_volume）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evsv3_volume)
- [V3云硬盘快照（huaweicloud_evsv3_snapshot）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evsv3_snapshot)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_evsv3_volume

data.huaweicloud_images_images
    └── huaweicloud_evsv3_volume

huaweicloud_evsv3_volume
    └── huaweicloud_evsv3_snapshot
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询当前区域下可用的可用区列表：

```hcl
# 查询当前区域下可用的可用区列表
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 该数据源无需额外参数，将自动查询当前区域下的可用区信息，其返回的 `names` 列表可用于为云硬盘指定可用区。

### 3. 查询镜像列表

在TF文件（如main.tf）中添加以下脚本以查询用于创建云硬盘的镜像：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询符合条件的镜像列表
variable "volume_image_id" {
  description = "The ID of the image used to create the volume, if not specified, the first available image matching the criteria will be used"
  type        = string
  default     = ""
}

variable "volume_image_visibility" {
  description = "The visibility of the volume image"
  type        = string
  default     = "public"
}

variable "volume_image_os" {
  description = "The OS of the volume image"
  type        = string
  default     = "Ubuntu"
}

data "huaweicloud_images_images" "test" {
  count = var.volume_image_id == "" ? 1 : 0

  visibility = var.volume_image_visibility
  os         = var.volume_image_os
}
```

**参数说明**：
- **count**：当未通过 `volume_image_id` 指定镜像时，才查询镜像列表，通过引用输入变量 volume_image_id 进行赋值
- **visibility**：镜像的可见性，通过引用输入变量 volume_image_visibility 进行赋值
- **os**：镜像的操作系统类型，通过引用输入变量 volume_image_os 进行赋值

### 4. 创建V3云硬盘

在TF文件（如main.tf）中添加以下脚本以创建V3云硬盘：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建V3云硬盘
variable "volume_type" {
  description = "The type of the volume"
  type        = string
  default     = "GPSSD"
}

variable "volume_availability_zone" {
  description = "The availability zone for the volume"
  type        = string
  default     = ""
  nullable    = false
}

variable "volume_description" {
  description = "The description of the volume"
  type        = string
  default     = ""
}

variable "volume_metadata" {
  description = "The metadata of the volume"
  type        = map(string)
  default     = {}
}

variable "volume_multiattach" {
  description = "The volume is shared volume or not"
  type        = bool
  default     = false
}

variable "volume_name" {
  description = "The name of the volume"
  type        = string
}

variable "volume_size" {
  description = "The size of the volume"
  type        = number
  default     = 40
}

variable "volume_tags" {
  description = "The tags of the volume"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_evsv3_volume" "test" {
  region            = var.region_name
  volume_type       = var.volume_type
  availability_zone = var.volume_availability_zone != "" ? var.volume_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  description       = var.volume_description
  image_id          = var.volume_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, "") : var.volume_image_id
  metadata          = var.volume_metadata
  multiattach       = var.volume_multiattach
  name              = var.volume_name
  size              = var.volume_size
  tags              = var.volume_tags
}
```

**参数说明**：
- **region**：资源所在的区域，通过引用输入变量 region_name 进行赋值
- **volume_type**：云硬盘的类型，通过引用输入变量 volume_type 进行赋值
- **availability_zone**：云硬盘所在的可用区，未指定时取可用区列表中的第一个可用区
- **description**：云硬盘的描述信息，通过引用输入变量 volume_description 进行赋值
- **image_id**：云硬盘使用的镜像ID，未指定时取镜像列表中的第一个镜像ID
- **metadata**：云硬盘的元数据，通过引用输入变量 volume_metadata 进行赋值
- **multiattach**：云硬盘是否为共享盘，通过引用输入变量 volume_multiattach 进行赋值
- **name**：云硬盘的名称，通过引用输入变量 volume_name 进行赋值
- **size**：云硬盘的大小，通过引用输入变量 volume_size 进行赋值
- **tags**：云硬盘的标签，通过引用输入变量 volume_tags 进行赋值

### 5. 创建V3云硬盘快照

在TF文件（如main.tf）中添加以下脚本以基于上述云硬盘创建快照：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建V3云硬盘快照
variable "snapshot_name" {
  description = "The name of the snapshot"
  type        = string
}

variable "snapshot_metadata" {
  description = "The metadata information of the snapshot"
  type        = map(string)
  default     = {}
}

variable "snapshot_description" {
  description = "The description of the snapshot"
  type        = string
  default     = ""
}

resource "huaweicloud_evsv3_snapshot" "test" {
  region      = var.region_name
  volume_id   = huaweicloud_evsv3_volume.test.id
  name        = var.snapshot_name
  metadata    = var.snapshot_metadata
  description = var.snapshot_description
}
```

**参数说明**：
- **region**：资源所在的区域，通过引用输入变量 region_name 进行赋值
- **volume_id**：快照所属的云硬盘ID，引用前一步创建的云硬盘ID进行赋值
- **name**：快照的名称，通过引用输入变量 snapshot_name 进行赋值
- **metadata**：快照的元数据，通过引用输入变量 snapshot_metadata 进行赋值
- **description**：快照的描述信息，通过引用输入变量 snapshot_description 进行赋值

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
volume_type        = "GPSSD"
volume_description = "Created by terraform"
volume_name        = "tf_test_volume"

volume_metadata = {
  test = "terraform volume"
}

volume_tags = {
  foo = "bar"
  key = "value"
}

snapshot_name        = "tf_test_snapshot"
snapshot_description = "Created by terraform"

snapshot_metadata = {
  test = "terraform snapshot"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="volume_name=my-volume"`
2. 环境变量：`export TF_VAR_volume_name=my-volume`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建V3云硬盘与快照
4. 运行 `terraform show` 查看已创建的V3云硬盘与快照

## 参考信息

- [华为云EVS产品文档](https://support.huaweicloud.com/evs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EVS V3云硬盘与快照最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/evs/v3-volume-snapshot)
