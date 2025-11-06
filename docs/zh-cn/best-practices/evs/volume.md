# 部署云硬盘

## 应用场景

云硬盘（Elastic Volume Service，EVS）是华为云提供的高性能、高可靠、可扩展的块存储服务，为ECS实例提供持久化存储。EVS支持多种存储类型，包括SSD、SAS、SATA等，满足不同业务场景的存储需求。

云硬盘是EVS服务的核心资源，提供持久化存储能力，支持多种存储类型和性能配置。通过云硬盘，企业可以为ECS实例提供可靠的数据存储，支持数据备份、快照、扩容等高级功能。云硬盘支持多种设备类型和性能配置，满足不同应用场景的存储需求。本最佳实践将介绍如何使用Terraform自动化部署云硬盘，包括可用区选择、存储类型配置和性能参数设置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [云硬盘资源（huaweicloud_evs_volume）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_evs_volume.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建云硬盘：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建云硬盘
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 无需额外参数，数据源会自动获取当前region下的所有可用区信息

### 3. 创建云硬盘

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云硬盘资源：

```hcl
variable "volume_availability_zone" {
  description = "云硬盘可用区"
  type        = string
  default     = ""
  nullable    = false
}

variable "volume_name" {
  description = "云硬盘名称"
  type        = string
}

variable "volume_type" {
  description = "云硬盘类型"
  type        = string
  default     = "SSD"
}

variable "voulme_size" {
  description = "云硬盘大小"
  type        = number
  default     = 40
}

variable "volume_description" {
  description = "云硬盘描述"
  type        = string
  default     = ""
}

variable "volume_multiattach" {
  description = "云硬盘是否为共享盘"
  type        = bool
  default     = false
}

variable "volume_iops" {
  description = "云硬盘IOPS"
  type        = number
  default     = null
}

variable "volume_throughput" {
  description = "云硬盘吞吐量"
  type        = number
  default     = null
}

variable "volume_device_type" {
  description = "云硬盘设备类型"
  type        = string
  default     = "VBD"
}

variable "enterprise_project_id" {
  description = "云硬盘企业项目ID"
  type        = string
  default     = null
}

variable "volume_tags" {
  description = "云硬盘标签"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "云硬盘计费模式"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "云硬盘计费周期单位"
  type        = string
  default     = null
}

variable "period" {
  description = "云硬盘计费周期"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "云硬盘是否自动续费"
  type        = string
  default     = "false"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云硬盘资源
resource "huaweicloud_evs_volume" "test" {
  availability_zone     = var.volume_availability_zone != "" ? var.volume_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  name                  = var.volume_name
  volume_type           = var.volume_type
  size                  = var.voulme_size
  description           = var.volume_description
  multiattach           = var.volume_multiattach
  iops                  = var.volume_iops
  throughput            = var.volume_throughput
  device_type           = var.volume_device_type
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.volume_tags
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew
}
```

**参数说明**：
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区数据源的第一个结果
- **name**：云硬盘名称，通过引用输入变量volume_name进行赋值
- **volume_type**：云硬盘类型，通过引用输入变量volume_type进行赋值
- **size**：云硬盘大小，通过引用输入变量voulme_size进行赋值
- **description**：云硬盘描述，通过引用输入变量volume_description进行赋值
- **multiattach**：是否为共享盘，通过引用输入变量volume_multiattach进行赋值
- **iops**：IOPS性能，通过引用输入变量volume_iops进行赋值
- **throughput**：吞吐量性能，通过引用输入变量volume_throughput进行赋值
- **device_type**：设备类型，通过引用输入变量volume_device_type进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值
- **tags**：标签，通过引用输入变量volume_tags进行赋值
- **charging_mode**：计费模式，通过引用输入变量charging_mode进行赋值
- **period_unit**：计费周期单位，通过引用输入变量period_unit进行赋值
- **period**：计费周期，通过引用输入变量period进行赋值
- **auto_renew**：自动续费，通过引用输入变量auto_renew进行赋值

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 云硬盘配置
volume_name        = "tf_test_vplume"
volume_type        = "SSD"
voulme_size        = 40
volume_description = "terraform test"
volume_device_type = "VBD"
volume_tags        = {
  foo = "bar"
  key = "value"
}
charging_mode      = "postPaid"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="volume_name=my-volume" -var="volume_type=SSD"`
2. 环境变量：`export TF_VAR_volume_name=my-volume`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建云硬盘
4. 运行 `terraform show` 查看已创建的云硬盘

## 参考信息

- [华为云EVS产品文档](https://support.huaweicloud.com/evs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EVS云硬盘最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/evs/volume)
