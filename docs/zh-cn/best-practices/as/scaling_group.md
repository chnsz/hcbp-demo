# 部署弹性伸缩组

## 应用场景

华为云弹性伸缩服务（Auto Scaling）是一种自动调整计算资源的服务，能够根据业务需求和策略，自动调整弹性计算实例的数量。弹性伸缩组是弹性伸缩服务的核心资源，用于管理一组具有相同配置和规则的弹性计算实例。通过创建弹性伸缩组，可以自动管理实例数量的增减，实现资源的弹性扩展和收缩，满足业务负载变化的需求。本最佳实践将介绍如何使用Terraform自动化部署一个弹性伸缩组，包括可用区、实例规格、镜像的查询，以及安全组、密钥对、AS配置、VPC网络和弹性伸缩组的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [计算规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [密钥对资源（huaweicloud_kps_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [AS配置资源（huaweicloud_as_configuration）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_configuration)
- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [AS组资源（huaweicloud_as_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_group)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── data.huaweicloud_images_images
            └── huaweicloud_as_configuration

huaweicloud_networking_secgroup
    └── huaweicloud_as_configuration
        └── huaweicloud_as_group

huaweicloud_kps_keypair
    └── huaweicloud_as_configuration
        └── huaweicloud_as_group

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_as_group
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询弹性伸缩组资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建弹性伸缩组相关资源：

```hcl
variable "availability_zone" {
  description = "弹性伸缩配置所属的可用区"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建弹性伸缩组相关资源
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询，仅当 `var.availability_zone` 为空时创建数据源（即执行可用区列表查询）

### 3. 通过数据源查询弹性伸缩组资源创建所需的实例规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的实例规格：

```hcl
variable "configuration_flavor_id" {
  description = "伸缩配置的规格ID"
  type        = string
  default     = ""
}

variable "configuration_flavor_performance_type" {
  description = "伸缩配置的性能类型"
  type        = string
  default     = "normal"
}

variable "configuration_flavor_cpu_core_count" {
  description = "伸缩配置的CPU核心数"
  type        = number
  default     = 2
}

variable "configuration_flavor_memory_size" {
  description = "伸缩配置的内存大小"
  type        = number
  default     = 4
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的实例规格信息，用于创建弹性伸缩组相关资源
data "huaweicloud_compute_flavors" "test" {
  count = var.configuration_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  performance_type  = var.configuration_flavor_performance_type
  cpu_core_count    = var.configuration_flavor_cpu_core_count
  memory_size       = var.configuration_flavor_memory_size
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行实例规格列表查询，仅当 `var.configuration_flavor_id` 为空时创建数据源（即执行实例规格列表查询）
- **availability_zone**：实例规格所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区
- **performance_type**：性能类型，通过输入变量 `configuration_flavor_performance_type` 进行赋值，默认为"normal"表示标准型
- **cpu_core_count**：CPU核心数，通过输入变量 `configuration_flavor_cpu_core_count` 进行赋值
- **memory_size**：内存大小（GB），通过输入变量 `configuration_flavor_memory_size` 进行赋值

### 4. 通过数据源查询弹性伸缩组资源创建所需的镜像

在TF文件中添加以下脚本以告知Terraform查询符合条件的镜像：

```hcl
variable "configuration_image_id" {
  description = "伸缩配置的镜像ID"
  type        = string
  default     = ""
}

variable "configuration_image_visibility" {
  description = "镜像的可见性"
  type        = string
  default     = "public"
}

variable "configuration_image_os" {
  description = "镜像的操作系统"
  type        = string
  default     = "Ubuntu"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的镜像信息，用于创建弹性伸缩组相关资源
data "huaweicloud_images_images" "test" {
  count = var.configuration_image_id == "" ? 1 : 0

  flavor_id  = var.configuration_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null) : var.configuration_flavor_id
  visibility = var.configuration_image_visibility
  os         = var.configuration_image_os
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行镜像列表查询，仅当 `var.configuration_image_id` 为空时创建数据源（即执行镜像列表查询）
- **flavor_id**：镜像所支持的规格ID，如果指定了规格ID则使用该值，否则使用实例规格列表查询数据源的第一个规格ID
- **visibility**：镜像可见性，通过输入变量 `configuration_image_visibility` 进行赋值，默认为"public"表示公共镜像
- **os**：操作系统类型，通过输入变量 `configuration_image_os` 进行赋值，默认为"Ubuntu"

### 5. 创建安全组资源

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "安全组的名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于为弹性伸缩组提供安全防护
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组的名称，通过引用输入变量 `security_group_name` 进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 6. 创建密钥对资源

在TF文件中添加以下脚本以告知Terraform创建密钥对资源：

```hcl
variable "keypair_name" {
  description = "密钥对的名称"
  type        = string
}

variable "keypair_public_key" {
  description = "用于SSH访问的公钥"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建密钥对资源，用于访问弹性伸缩实例
resource "huaweicloud_kps_keypair" "test" {
  name       = var.keypair_name
  public_key = var.keypair_public_key != "" ? var.keypair_public_key : null
}
```

**参数说明**：
- **name**：密钥对的名称，通过引用输入变量 `keypair_name` 进行赋值
- **public_key**：密钥对的公钥，如果指定了公钥则使用该值，否则设置为null（系统将自动生成密钥对）

### 7. 创建AS配置资源

在TF文件中添加以下脚本以告知Terraform创建AS配置资源：

```hcl
variable "configuration_name" {
  description = "伸缩配置的名称"
  type        = string
}

variable "configuration_metadata" {
  description = "伸缩配置实例的元数据"
  type        = map(string)
}

variable "configuration_user_data" {
  description = "伸缩配置实例初始化的用户数据脚本"
  type        = string
}

variable "configuration_disks" {
  description = "伸缩配置实例的磁盘配置"
  type = list(object({
    size        = number
    volume_type = string
    disk_type   = string
  }))

  nullable = false
}

variable "configuration_public_eip_settings" {
  description = "伸缩配置实例的公网IP设置"
  type = list(object({
    ip_type = string
    bandwidth = object({
      size          = number
      share_type    = string
      charging_mode = string
    })
  }))

  nullable = false
  default  = []
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AS配置资源，用于定义弹性伸缩实例的模板
resource "huaweicloud_as_configuration" "test" {
  scaling_configuration_name = var.configuration_name

  instance_config {
    image              = var.configuration_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, null) : var.configuration_image_id
    flavor             = var.configuration_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null) : var.configuration_flavor_id
    key_name           = huaweicloud_kps_keypair.test.id
    security_group_ids = [huaweicloud_networking_secgroup.test.id]

    metadata  = var.configuration_metadata
    user_data = var.configuration_user_data

    dynamic "disk" {
      for_each = var.configuration_disks

      content {
        size        = disk.value["size"]
        volume_type = disk.value["volume_type"]
        disk_type   = disk.value["disk_type"]
      }
    }

    dynamic "public_ip" {
      for_each = var.configuration_public_eip_settings

      content {
        eip {
          ip_type = public_ip.value.ip_type

          bandwidth {
            size          = public_ip.value.bandwidth.size
            share_type    = public_ip.value.bandwidth.share_type
            charging_mode = public_ip.value.bandwidth.charging_mode
          }
        }
      }
    }
  }
}
```

**参数说明**：
- **scaling_configuration_name**：AS配置的名称，通过引用输入变量 `configuration_name` 进行赋值
- **instance_config**：实例配置块，定义弹性伸缩实例的配置
  - **image**：实例镜像ID，如果指定了镜像ID则使用该值，否则使用镜像列表查询数据源的第一个镜像ID
  - **flavor**：实例规格ID，如果指定了规格ID则使用该值，否则使用实例规格列表查询数据源的第一个规格ID
  - **key_name**：密钥对ID，引用密钥对资源（huaweicloud_kps_keypair.test）的ID进行赋值
  - **security_group_ids**：安全组ID列表，引用安全组资源（huaweicloud_networking_secgroup.test）的ID进行赋值
  - **metadata**：实例元数据，通过输入变量 `configuration_metadata` 进行赋值，用于在实例启动时注入自定义数据
  - **user_data**：用户数据脚本，通过输入变量 `configuration_user_data` 进行赋值，用于在实例启动时执行初始化脚本
  - **disk**：磁盘配置块，通过动态块（dynamic block）根据输入变量 `configuration_disks` 创建多个磁盘配置
    - **size**：磁盘大小（GB），通过输入变量 `configuration_disks` 中的 `size` 进行赋值
    - **volume_type**：磁盘类型，通过输入变量 `configuration_disks` 中的 `volume_type` 进行赋值
    - **disk_type**：磁盘类型，通过输入变量 `configuration_disks` 中的 `disk_type` 进行赋值
  - **public_ip**：公网IP配置块，通过动态块（dynamic block）根据输入变量 `configuration_public_eip_settings` 创建公网IP配置
    - **eip**：弹性公网IP配置块
      - **ip_type**：IP类型，通过输入变量 `configuration_public_eip_settings` 中的 `ip_type` 进行赋值
      - **bandwidth**：带宽配置块
        - **size**：带宽大小（Mbps），通过输入变量 `configuration_public_eip_settings` 中的 `bandwidth.size` 进行赋值
        - **share_type**：共享类型，通过输入变量 `configuration_public_eip_settings` 中的 `bandwidth.share_type` 进行赋值
        - **charging_mode**：计费模式，通过输入变量 `configuration_public_eip_settings` 中的 `bandwidth.charging_mode` 进行赋值

> 注意：磁盘配置中必须包含至少一个系统盘（disk_type为"SYS"），其他磁盘为数据盘。公网IP配置为可选配置，如果不需要公网IP，可以不配置 `configuration_public_eip_settings` 或设置为空列表。

### 8. 创建VPC资源（可选）

在TF文件中添加以下脚本以告知Terraform创建VPC资源（如果未指定VPC ID）：

```hcl
variable "scaling_group_vpc_id" {
  description = "VPC的ID"
  type        = string
  default     = ""
}

variable "scaling_group_vpc_name" {
  description = "VPC的名称"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.scaling_group_vpc_id == "" && var.scaling_group_vpc_name != ""
    error_message = "当'scaling_group_vpc_id'为空时，'scaling_group_vpc_name'不允许为空"
  }
}

variable "scaling_group_vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于为弹性伸缩组提供网络环境
resource "huaweicloud_vpc" "test" {
  count = var.scaling_group_vpc_id == "" && var.scaling_group_subnet_id == "" ? 1 : 0

  name = var.scaling_group_vpc_name
  cidr = var.scaling_group_vpc_cidr
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建VPC资源，仅当 `var.scaling_group_vpc_id` 和 `var.scaling_group_subnet_id` 都为空时创建VPC资源
- **name**：VPC的名称，通过引用输入变量 `scaling_group_vpc_name` 进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量 `scaling_group_vpc_cidr` 进行赋值，默认为"192.168.0.0/16"

### 9. 创建VPC子网资源（可选）

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源（如果未指定子网ID）：

```hcl
variable "scaling_group_subnet_id" {
  description = "子网的ID"
  type        = string
  default     = ""
}

variable "scaling_group_subnet_name" {
  description = "子网的名称"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.scaling_group_subnet_id == "" && var.scaling_group_subnet_name != ""
    error_message = "当'scaling_group_subnet_id'为空时，'scaling_group_subnet_name'不允许为空"
  }
}

variable "scaling_group_subnet_cidr" {
  description = "子网的CIDR块"
  type        = string
  default     = ""
}

variable "scaling_group_subnet_gateway_ip" {
  description = "子网的网关IP"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于为弹性伸缩组提供网络环境
resource "huaweicloud_vpc_subnet" "test" {
  count = var.scaling_group_subnet_id == "" ? 1 : 0

  vpc_id     = var.scaling_group_vpc_id == "" ? huaweicloud_vpc.test[0].id : var.scaling_group_vpc_id
  name       = var.scaling_group_subnet_name
  cidr       = var.scaling_group_subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test[0].cidr, 8, 0) : var.scaling_group_subnet_cidr
  gateway_ip = var.scaling_group_subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test[0].cidr, 8, 0), 1) : var.scaling_group_subnet_gateway_ip
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建VPC子网资源，仅当 `var.scaling_group_subnet_id` 为空时创建VPC子网资源
- **vpc_id**：子网所属的VPC ID，如果指定了VPC ID则使用该值，否则引用VPC资源（huaweicloud_vpc.test[0]）的ID进行赋值
- **name**：子网的名称，通过引用输入变量 `scaling_group_subnet_name` 进行赋值
- **cidr**：子网的CIDR块，如果指定了子网CIDR则使用该值，否则基于VPC的CIDR块通过 `cidrsubnet` 函数自动计算
- **gateway_ip**：子网的网关IP，如果指定了网关IP则使用该值，否则基于子网CIDR通过 `cidrhost` 函数自动计算

### 10. 创建AS组资源

在TF文件中添加以下脚本以告知Terraform创建AS组资源：

```hcl
variable "scaling_group_name" {
  description = "弹性伸缩组的名称"
  type        = string
}

variable "scaling_group_desire_instance_number" {
  description = "期望的实例数量"
  type        = number
  default     = 2
}

variable "scaling_group_min_instance_number" {
  description = "最小的实例数量"
  type        = number
  default     = 0
}

variable "scaling_group_max_instance_number" {
  description = "最大的实例数量"
  type        = number
  default     = 10
}

variable "is_delete_scaling_group_publicip" {
  description = "删除弹性伸缩组时是否删除弹性伸缩实例的公网IP地址"
  type        = bool
  default     = true
}

variable "is_delete_scaling_group_instances" {
  description = "删除弹性伸缩组时是否删除弹性伸缩实例"
  type        = string
  default     = "yes"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AS组资源，用于管理弹性伸缩实例
resource "huaweicloud_as_group" "test" {
  scaling_group_name       = var.scaling_group_name
  scaling_configuration_id = huaweicloud_as_configuration.test.id
  desire_instance_number   = var.scaling_group_desire_instance_number
  min_instance_number      = var.scaling_group_min_instance_number
  max_instance_number      = var.scaling_group_max_instance_number
  vpc_id                   = var.scaling_group_vpc_id == "" ? huaweicloud_vpc.test[0].id : var.scaling_group_vpc_id
  delete_publicip          = var.is_delete_scaling_group_publicip
  delete_instances         = var.is_delete_scaling_group_instances

  networks {
    id = var.scaling_group_subnet_id == "" ? huaweicloud_vpc_subnet.test[0].id : var.scaling_group_subnet_id
  }
}
```

**参数说明**：
- **scaling_group_name**：AS组的名称，通过引用输入变量 `scaling_group_name` 进行赋值
- **scaling_configuration_id**：AS配置ID，引用AS配置资源（huaweicloud_as_configuration.test）的ID进行赋值
- **desire_instance_number**：期望的弹性伸缩实例数量，通过引用输入变量 `scaling_group_desire_instance_number` 进行赋值，默认为2
- **min_instance_number**：最小的弹性伸缩实例数量，通过引用输入变量 `scaling_group_min_instance_number` 进行赋值，默认为0
- **max_instance_number**：最大的弹性伸缩实例数量，通过引用输入变量 `scaling_group_max_instance_number` 进行赋值，默认为10
- **vpc_id**：VPC ID，如果指定了VPC ID则使用该值，否则引用VPC资源（huaweicloud_vpc.test[0]）的ID进行赋值
- **delete_publicip**：删除AS组时是否删除弹性伸缩实例的公网IP地址，通过引用输入变量 `is_delete_scaling_group_publicip` 进行赋值，默认为true
- **delete_instances**：删除AS组时是否删除弹性伸缩实例，通过引用输入变量 `is_delete_scaling_group_instances` 进行赋值，值为"yes"或"no"，默认为"yes"
- **networks**：网络配置块，定义AS组使用的子网
  - **id**：子网ID，如果指定了子网ID则使用该值，否则引用VPC子网资源（huaweicloud_vpc_subnet.test[0]）的ID进行赋值

> 注意：`min_instance_number`、`desire_instance_number` 和 `max_instance_number` 必须满足关系：`min_instance_number <= desire_instance_number <= max_instance_number`。

### 11. 预设资源部署所需的入参

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
security_group_name     = "tf_test_secgroup_demo"
keypair_name            = "tf_test_keypair_demo"
configuration_name      = "tf_test_as_configuration"
configuration_metadata  = {
  some_key = "some_value"
}
configuration_user_data = <<EOT
# !/bin/sh
echo "Hello World! The time is now $(date -R)!" | tee /root/output.txt
EOT

configuration_disks = [
  {
    size        = 40
    volume_type = "SSD"
    disk_type   = "SYS"
  }
]

configuration_public_eip_settings = [
  {
    ip_type   = "5_bgp"
    bandwidth = {
      size          = 10
      share_type    = "PER"
      charging_mode = "traffic"
    }
  }
]

scaling_group_vpc_name    = "tf_test_vpc_demo"
scaling_group_subnet_name = "tf_test_subnet_demo"
scaling_group_name        = "tf_test_scaling_group_demo"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="security_group_name=my-secgroup" -var="keypair_name=my-keypair"`
2. 环境变量：`export TF_VAR_security_group_name=my-secgroup`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 12. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建弹性伸缩组
4. 运行 `terraform show` 查看已创建的弹性伸缩组

## 参考信息

- [华为云弹性伸缩产品文档](https://support.huaweicloud.com/as/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [AS弹性伸缩组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/as/scaling-group)
