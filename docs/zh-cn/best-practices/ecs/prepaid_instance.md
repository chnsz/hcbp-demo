# 部署包周期实例

## 应用场景

弹性云服务器（Elastic Cloud Server，ECS）是由CPU、内存、操作系统、云硬盘组成的基础的计算组件。弹性云服务器创建成功后，您就可以像使用自己的本地PC或物理服务器一样，在云上使用弹性云服务器。华为云提供了多种类型的弹性云服务器，可满足不同的使用场景。包周期（PrePaid）实例是一种预付费的计费模式，相比按需付费（PostPaid）实例，包周期实例通常具有更优惠的价格，适合长期稳定运行的工作负载。在创建之前，您需要根据实际的应用场景确认弹性云服务器的规格类型、镜像类型、磁盘种类等参数，并选择合适的网络参数和安全组规则。本最佳实践将介绍如何使用Terraform自动化部署一个包周期ECS实例，包括VPC、子网和安全组的创建，以及包周期计费模式的配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_compute_flavors
    │   └── data.huaweicloud_images_images
    │       └── huaweicloud_compute_instance
    └── huaweicloud_vpc_subnet
        └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询ECS实例资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例：

```hcl
variable "availability_zone" {
  description = "ECS实例所属的可用区"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建ECS实例
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询可用区列表，仅当 `var.availability_zone` 为空时查询可用区列表

### 3. 通过数据源查询ECS实例资源创建所需的规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的ECS规格：

```hcl
variable "instance_flavor_id" {
  description = "ECS实例的规格ID"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_performance_type" {
  description = "ECS实例规格的性能类型"
  type        = string
  default     = "normal"
}

variable "instance_cpu_core_count" {
  description = "ECS实例的CPU核心数"
  type        = number
  default     = 2
}

variable "instance_memory_size" {
  description = "ECS实例的内存大小（GB）"
  type        = number
  default     = 4
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的ECS规格信息，用于创建ECS实例
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  performance_type  = var.instance_performance_type
  cpu_core_count    = var.instance_cpu_core_count
  memory_size       = var.instance_memory_size
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询实例规格信息，仅当 `var.instance_flavor_id` 为空时查询实例规格信息
- **availability_zone**：规格所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区
- **performance_type**：规格的性能类型，通过引用输入变量 `instance_performance_type` 进行赋值，默认为"normal"表示标准类型
- **cpu_core_count**：CPU核心数，通过引用输入变量 `instance_cpu_core_count` 进行赋值，默认为2核
- **memory_size**：内存大小（GB），通过引用输入变量 `instance_memory_size` 进行赋值，默认为4GB

### 4. 通过数据源查询ECS实例资源创建所需的镜像

在TF文件中添加以下脚本以告知Terraform查询符合条件的镜像：

```hcl
variable "instance_image_id" {
  description = "ECS实例的镜像ID"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_image_visibility" {
  description = "ECS实例镜像的可见性"
  type        = string
  default     = "public"
}

variable "instance_image_os" {
  description = "ECS实例镜像的操作系统"
  type        = string
  default     = "Ubuntu"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的IMS镜像信息，用于创建ECS实例
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询镜像信息，仅当 `var.instance_image_id` 为空时查询镜像信息
- **flavor_id**：镜像支持的规格ID，如果指定了实例规格ID则使用该值，否则根据计算规格列表查询数据源的返回结果进行赋值
- **visibility**：镜像的可见性，通过引用输入变量 `instance_image_visibility` 进行赋值，默认为"public"表示公共镜像
- **os**：镜像的操作系统类型，通过引用输入变量 `instance_image_os` 进行赋值，默认为"Ubuntu"操作系统

### 5. 创建VPC资源

在TF文件中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "VPC的名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署ECS实例
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC的名称，通过引用输入变量 `vpc_name` 进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量 `vpc_cidr` 进行赋值，默认为"192.168.0.0/16"网段

### 6. 创建VPC子网资源

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "子网的名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网的CIDR块"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "子网的网关IP地址"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署ECS实例
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量 `subnet_name` 进行赋值
- **cidr**：子网的CIDR网段，如果指定了子网CIDR则使用该值，否则使用cidrsubnet函数从VPC的CIDR网段中划分一个子网段
- **gateway_ip**：子网的网关IP，如果指定了网关IP则使用该值，否则使用cidrhost函数从子网网段中获取第一个IP地址作为网关IP

### 7. 创建安全组资源

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_ids" {
  description = "ECS实例的安全组ID列表"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "security_group_name" {
  description = "安全组的名称"
  type        = string
  default     = ""

  validation {
    condition     = !(length(var.security_group_ids) < 1 && var.security_group_name == "")
    error_message = "如果未设置安全组ID列表，则安全组名称不能为空"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署ECS实例
resource "huaweicloud_networking_secgroup" "test" {
  count = length(var.security_group_ids) < 1 ? 1 : 0

  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建安全组资源，仅当安全组ID列表为空时创建安全组资源
- **name**：安全组的名称，通过引用输入变量 `security_group_name` 进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 8. 创建ECS实例资源

在TF文件中添加以下脚本以告知Terraform创建包周期ECS实例资源：

```hcl
variable "instance_name" {
  description = "ECS实例的名称"
  type        = string
}

variable "instance_admin_password" {
  description = "ECS实例的登录密码"
  type        = string
  sensitive   = true
}

variable "instance_description" {
  description = "ECS实例的描述"
  type        = string
  default     = null
}

variable "instance_system_disk_type" {
  description = "ECS实例系统盘的类型"
  type        = string
  default     = null
}

variable "instance_system_disk_size" {
  description = "ECS实例系统盘的大小（GB）"
  type        = number
  default     = null
}

variable "instance_system_disk_iops" {
  description = "ECS实例系统盘的IOPS"
  type        = number
  default     = null
}

variable "instance_system_disk_throughput" {
  description = "ECS实例系统盘的吞吐量"
  type        = number
  default     = null
}

variable "instance_system_disk_dss_pool_id" {
  description = "ECS实例系统盘的DSS池ID"
  type        = string
  default     = null
}

variable "instance_metadata" {
  description = "ECS实例的元数据键值对"
  type        = map(string)
  default     = null
}

variable "instance_tags" {
  description = "与ECS实例关联的键值对"
  type        = map(string)
  default     = null
}

variable "enterprise_project_id" {
  description = "ECS实例所属的企业项目ID"
  type        = string
  default     = null
}

variable "instance_eip_id" {
  description = "与ECS实例关联的EIP ID"
  type        = string
  default     = null
}

variable "instance_eip_type" {
  description = "创建并与ECS实例关联的EIP类型"
  type        = string
  default     = null
}

variable "instance_bandwidth" {
  description = "ECS实例的带宽配置"
  type = object({
    share_type   = string
    id           = optional(string, null)
    size         = optional(number, null)
    charge_mode  = optional(string, null)
    extend_param = optional(map(string), null)
  })
  default = null
}

variable "period_unit" {
  description = "ECS实例的计费周期单位"
  type        = string
  default     = "month"
}

variable "period" {
  description = "ECS实例的计费周期"
  type        = number
  default     = 1
}

variable "auto_renew" {
  description = "是否启用ECS实例的自动续费"
  type        = string
  default     = "false"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建包周期ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  name                    = var.instance_name
  image_id                = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
  flavor_id               = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  security_group_ids      = length(var.security_group_ids) != 0 ? var.security_group_ids : huaweicloud_networking_secgroup.test[*].id
  availability_zone       = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  admin_pass              = var.instance_admin_password
  description             = var.instance_description
  system_disk_type        = var.instance_system_disk_type
  system_disk_size        = var.instance_system_disk_size
  system_disk_iops        = var.instance_system_disk_iops
  system_disk_throughput  = var.instance_system_disk_throughput
  system_disk_dss_pool_id = var.instance_system_disk_dss_pool_id
  metadata                = var.instance_metadata
  tags                    = var.instance_tags
  enterprise_project_id   = var.enterprise_project_id
  eip_id                  = var.instance_eip_id
  eip_type                = var.instance_eip_type

  dynamic "bandwidth" {
    for_each = var.instance_bandwidth == null ? [] : [var.instance_bandwidth]

    content {
      share_type   = bandwidth.value.share_type
      id           = bandwidth.value.id
      size         = bandwidth.value.size
      charge_mode  = bandwidth.value.charge_mode
      extend_param = bandwidth.value.extend_param
    }
  }

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  charging_mode = "prePaid"
  period_unit   = var.period_unit
  period        = var.period
  auto_renew    = var.auto_renew
}
```

**参数说明**：
- **name**：ECS实例的名称，通过引用输入变量 `instance_name` 进行赋值
- **image_id**：ECS实例所使用镜像的ID，如果指定了镜像ID则使用该值，否则根据镜像列表查询数据源的返回结果进行赋值
- **flavor_id**：ECS实例所使用规格的ID，如果指定了实例规格ID则使用该值，否则根据计算规格列表查询数据源的返回结果进行赋值
- **security_group_ids**：与ECS实例关联的安全组ID列表，如果指定了安全组ID列表则使用该值，否则使用创建的安全组资源的ID
- **availability_zone**：ECS实例所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区
- **admin_pass**：ECS实例的管理员密码，通过引用输入变量 `instance_admin_password` 进行赋值
- **description**：ECS实例的描述，通过引用输入变量 `instance_description` 进行赋值
- **system_disk_type**：系统盘类型，通过引用输入变量 `instance_system_disk_type` 进行赋值
- **system_disk_size**：系统盘大小（GB），通过引用输入变量 `instance_system_disk_size` 进行赋值
- **system_disk_iops**：系统盘IOPS，通过引用输入变量 `instance_system_disk_iops` 进行赋值
- **system_disk_throughput**：系统盘吞吐量，通过引用输入变量 `instance_system_disk_throughput` 进行赋值
- **system_disk_dss_pool_id**：系统盘DSS池ID，通过引用输入变量 `instance_system_disk_dss_pool_id` 进行赋值
- **metadata**：ECS实例的元数据键值对，通过引用输入变量 `instance_metadata` 进行赋值
- **tags**：与ECS实例关联的标签键值对，通过引用输入变量 `instance_tags` 进行赋值
- **enterprise_project_id**：ECS实例所属的企业项目ID，通过引用输入变量 `enterprise_project_id` 进行赋值
- **eip_id**：与ECS实例关联的EIP ID，通过引用输入变量 `instance_eip_id` 进行赋值
- **eip_type**：创建并与ECS实例关联的EIP类型，通过引用输入变量 `instance_eip_type` 进行赋值
- **bandwidth**：带宽配置块（动态块），根据输入变量 `instance_bandwidth` 动态创建
  - **share_type**：带宽共享类型，通过引用输入变量中的带宽配置进行赋值
  - **id**：带宽ID，通过引用输入变量中的带宽配置进行赋值
  - **size**：带宽大小，通过引用输入变量中的带宽配置进行赋值
  - **charge_mode**：带宽计费模式，通过引用输入变量中的带宽配置进行赋值
  - **extend_param**：带宽扩展参数，通过引用输入变量中的带宽配置进行赋值
- **network**：网络配置块，指定ECS实例连接的网络
  - **uuid**：网络的唯一标识符，使用前面创建的子网资源的ID
- **charging_mode**：计费模式，设置为"prePaid"表示包周期（预付费）模式
- **period_unit**：计费周期单位，通过引用输入变量 `period_unit` 进行赋值，默认为"month"表示按月计费
- **period**：计费周期，通过引用输入变量 `period` 进行赋值，默认为1表示1个计费周期
- **auto_renew**：是否自动续费，通过引用输入变量 `auto_renew` 进行赋值，默认为"false"表示不自动续费

> 注意：包周期实例采用预付费模式，需要预先支付费用。计费周期单位可以是"month"（月）或"year"（年），计费周期可以是1-9（月）或1-3（年）。启用自动续费后，实例到期前会自动续费，避免因到期未续费导致服务中断。

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 网络资源配置
vpc_name                = "tf_test_vpc"
subnet_name             = "tf_test_subnet"
security_group_name     = "tf_test_security_group"

# ECS实例配置
instance_name           = "tf_test_prepaid_instace"
instance_admin_password = "YourPassword!"
instance_description    = "Creates a PrePaid ECS instance by Terraform"
instance_tags           = {
  owner = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="instance_name=my-instance"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建包周期ECS实例
4. 运行 `terraform show` 查看已创建的包周期ECS实例详情

## 参考信息

- [华为云ECS产品文档](https://support.huaweicloud.com/ecs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ECS包周期实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ecs/prepaid-instance)
