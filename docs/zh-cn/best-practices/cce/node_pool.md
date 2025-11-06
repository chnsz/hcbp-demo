# 部署节点池

## 应用场景

云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。节点池是CCE集群中用于管理一组具有相同配置的节点资源的集合，通过节点池可以统一管理节点的规格、镜像、存储等配置，并支持自动扩缩容功能。通过创建节点池，可以快速为CCE集群添加计算节点，实现容器工作负载的弹性扩展。本最佳实践将介绍如何使用Terraform自动化部署一个CCE节点池，包括可用区、实例规格的查询，以及VPC、子网、弹性公网IP、CCE集群、密钥对和节点池的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [计算规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [弹性公网IP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [CCE集群资源（huaweicloud_cce_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_cluster)
- [密钥对资源（huaweicloud_kps_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [CCE节点池资源（huaweicloud_cce_node_pool）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_node_pool)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_vpc_subnet
    └── data.huaweicloud_compute_flavors
        └── huaweicloud_cce_node_pool

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_cce_cluster
            └── huaweicloud_cce_node_pool

huaweicloud_vpc_eip
    └── huaweicloud_cce_cluster
        └── huaweicloud_cce_node_pool

huaweicloud_kps_keypair
    └── huaweicloud_cce_node_pool
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询节点池资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建节点池相关资源：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建节点池相关资源
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
此数据源无需额外参数，默认查询当前区域下所有可用的可用区信息。

### 3. 创建VPC资源（可选）

在TF文件中添加以下脚本以告知Terraform创建VPC资源（如果未指定VPC ID）：

```hcl
variable "vpc_id" {
  description = "VPC的ID"
  type        = string
  default     = ""
}

variable "vpc_name" {
  description = "VPC的名称"
  type        = string
  default     = ""

  validation {
    condition     = var.vpc_id != "" || var.vpc_name != ""
    error_message = "如果未提供vpc_id，则必须提供vpc_name"
  }
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于为节点池提供网络环境
resource "huaweicloud_vpc" "test" {
  count = var.vpc_id == "" && var.subnet_id == "" ? 1 : 0

  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建VPC资源，仅当 `var.vpc_id` 和 `var.subnet_id` 都为空时创建VPC资源
- **name**：VPC的名称，通过引用输入变量 `vpc_name` 进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量 `vpc_cidr` 进行赋值，默认为"192.168.0.0/16"

### 4. 创建VPC子网资源（可选）

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源（如果未指定子网ID）：

```hcl
variable "subnet_id" {
  description = "子网的ID"
  type        = string
  default     = ""
}

variable "subnet_name" {
  description = "子网的名称"
  type        = string
  default     = ""

  validation {
    condition     = var.subnet_id == "" || var.subnet_name == ""
    error_message = "如果未提供subnet_id，则必须提供subnet_name"
  }
}

variable "subnet_cidr" {
  description = "子网的CIDR块"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "子网的网关IP"
  type        = string
  default     = ""
}

variable "availability_zone" {
  description = "创建CCE集群的可用区"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于为节点池提供网络环境
resource "huaweicloud_vpc_subnet" "test" {
  count = var.subnet_id == "" ? 1 : 0

  vpc_id            = var.vpc_id != "" ? var.vpc_id : huaweicloud_vpc.test[0].id
  name              = var.subnet_name
  cidr              = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test[0].cidr, 4, 0)
  gateway_ip        = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : var.subnet_cidr != "" ? cidrhost(var.subnet_cidr, 1) : cidrhost(cidrsubnet(huaweicloud_vpc.test[0].cidr, 4, 0), 1)
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建VPC子网资源，仅当 `var.subnet_id` 为空时创建VPC子网资源
- **vpc_id**：子网所属的VPC ID，如果指定了VPC ID则使用该值，否则引用VPC资源（huaweicloud_vpc.test[0]）的ID进行赋值
- **name**：子网的名称，通过引用输入变量 `subnet_name` 进行赋值
- **cidr**：子网的CIDR块，如果指定了子网CIDR则使用该值，否则基于VPC的CIDR块通过 `cidrsubnet` 函数自动计算
- **gateway_ip**：子网的网关IP，如果指定了网关IP则使用该值，否则根据子网CIDR或自动计算的子网CIDR通过 `cidrhost` 函数自动计算
- **availability_zone**：子网所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区

### 5. 创建弹性公网IP资源（可选）

在TF文件中添加以下脚本以告知Terraform创建弹性公网IP资源（如果未指定EIP地址）：

```hcl
variable "eip_address" {
  description = "CCE集群的EIP地址"
  type        = string
  default     = ""
}

variable "eip_type" {
  description = "EIP的类型"
  type        = string
  default     = "5_bgp"
}

variable "bandwidth_name" {
  description = "带宽的名称"
  type        = string
  default     = ""
}

variable "bandwidth_size" {
  description = "带宽的大小"
  type        = number
  default     = 5
}

variable "bandwidth_share_type" {
  description = "带宽的共享类型"
  type        = string
  default     = "PER"
}

variable "bandwidth_charge_mode" {
  description = "带宽的计费模式"
  type        = string
  default     = "traffic"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源，用于为节点池提供公网访问能力
resource "huaweicloud_vpc_eip" "test" {
  count = var.eip_address == "" ? 1 : 0

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.bandwidth_name
    size        = var.bandwidth_size
    share_type  = var.bandwidth_share_type
    charge_mode = var.bandwidth_charge_mode
  }
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建弹性公网IP资源，仅当 `var.eip_address` 为空时创建弹性公网IP资源
- **publicip**：公网IP配置块
  - **type**：公网IP类型，通过引用输入变量 `eip_type` 进行赋值，默认为"5_bgp"表示全动态BGP
- **bandwidth**：带宽配置块
  - **name**：带宽的名称，通过引用输入变量 `bandwidth_name` 进行赋值
  - **size**：带宽大小（Mbps），通过引用输入变量 `bandwidth_size` 进行赋值，默认为5
  - **share_type**：带宽共享类型，通过引用输入变量 `bandwidth_share_type` 进行赋值，默认为"PER"表示独享
  - **charge_mode**：带宽计费模式，通过引用输入变量 `bandwidth_charge_mode` 进行赋值，默认为"traffic"表示按流量计费

### 6. 创建CCE集群资源

在TF文件中添加以下脚本以告知Terraform创建CCE集群资源：

```hcl
variable "cluster_name" {
  description = "CCE集群的名称"
  type        = string
  default     = ""
}

variable "cluster_flavor_id" {
  description = "CCE集群的规格ID"
  type        = string
  default     = "cce.s1.small"
}

variable "cluster_version" {
  description = "CCE集群的版本"
  type        = string
  default     = null
  nullable    = true
}

variable "cluster_type" {
  description = "CCE集群的类型"
  type        = string
  default     = "VirtualMachine"
}

variable "container_network_type" {
  description = "容器网络类型"
  type        = string
  default     = "overlay_l2"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE集群资源，用于部署和管理容器化应用
resource "huaweicloud_cce_cluster" "test" {
  name                   = var.cluster_name
  flavor_id              = var.cluster_flavor_id
  cluster_version        = var.cluster_version
  cluster_type           = var.cluster_type
  container_network_type = var.container_network_type
  vpc_id                 = var.vpc_id != "" ? var.vpc_id : huaweicloud_vpc.test[0].id
  subnet_id              = var.subnet_id != "" ? var.subnet_id : huaweicloud_vpc_subnet.test[0].id
  eip                    = var.eip_address != "" ? var.eip_address : huaweicloud_vpc_eip.test[0].address
}
```

**参数说明**：
- **name**：CCE集群的名称，通过引用输入变量 `cluster_name` 进行赋值
- **flavor_id**：CCE集群的规格ID，通过引用输入变量 `cluster_flavor_id` 进行赋值，默认为"cce.s1.small"表示小规格集群
- **cluster_version**：CCE集群的版本，通过引用输入变量 `cluster_version` 进行赋值，如果为null则使用最新版本
- **cluster_type**：CCE集群的类型，通过引用输入变量 `cluster_type` 进行赋值，默认为"VirtualMachine"表示虚拟机类型
- **container_network_type**：容器网络类型，通过引用输入变量 `container_network_type` 进行赋值，默认为"overlay_l2"表示L2网络
- **vpc_id**：VPC ID，如果指定了VPC ID则使用该值，否则引用VPC资源（huaweicloud_vpc.test[0]）的ID进行赋值
- **subnet_id**：子网ID，如果指定了子网ID则使用该值，否则引用VPC子网资源（huaweicloud_vpc_subnet.test[0]）的ID进行赋值
- **eip**：弹性公网IP地址，如果指定了EIP地址则使用该值，否则引用弹性公网IP资源（huaweicloud_vpc_eip.test[0]）的地址进行赋值

### 7. 通过数据源查询节点池资源创建所需的实例规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的实例规格：

```hcl
variable "node_performance_type" {
  description = "节点的性能类型"
  type        = string
  default     = "general"
}

variable "node_cpu_core_count" {
  description = "节点的CPU核心数"
  type        = number
  default     = 4
}

variable "node_memory_size" {
  description = "节点的内存大小"
  type        = number
  default     = 8
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的实例规格信息，用于创建节点池相关资源
data "huaweicloud_compute_flavors" "test" {
  performance_type  = var.node_performance_type
  cpu_core_count    = var.node_cpu_core_count
  memory_size       = var.node_memory_size
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
}
```

**参数说明**：
- **performance_type**：性能类型，通过引用输入变量 `node_performance_type` 进行赋值，默认为"general"表示通用型
- **cpu_core_count**：CPU核心数，通过引用输入变量 `node_cpu_core_count` 进行赋值，默认为4核
- **memory_size**：内存大小（GB），通过引用输入变量 `node_memory_size` 进行赋值，默认为8GB
- **availability_zone**：实例规格所在的可用区，使用可用区列表查询数据源的第一个可用区

### 8. 创建密钥对资源

在TF文件中添加以下脚本以告知Terraform创建密钥对资源：

```hcl
variable "keypair_name" {
  description = "密钥对的名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建密钥对资源，用于访问节点池中的节点
resource "huaweicloud_kps_keypair" "test" {
  name = var.keypair_name
}
```

**参数说明**：
- **name**：密钥对的名称，通过引用输入变量 `keypair_name` 进行赋值

### 9. 创建CCE节点池资源

在TF文件中添加以下脚本以告知Terraform创建CCE节点池资源：

```hcl
variable "node_pool_type" {
  description = "节点池的类型"
  type        = string
  default     = "vm"
}

variable "node_pool_name" {
  description = "节点池的名称"
  type        = string
}

variable "node_pool_os_type" {
  description = "节点池的操作系统类型"
  type        = string
  default     = "EulerOS 2.9"
}

variable "node_pool_initial_node_count" {
  description = "节点池的初始节点数量"
  type        = number
  default     = 2
}

variable "node_pool_min_node_count" {
  description = "节点池的最小节点数量"
  type        = number
  default     = 2
}

variable "node_pool_max_node_count" {
  description = "节点池的最大节点数量"
  type        = number
  default     = 10
}

variable "node_pool_scale_down_cooldown_time" {
  description = "节点池的缩容冷却时间"
  type        = number
  default     = 10
}

variable "node_pool_priority" {
  description = "节点池的优先级"
  type        = number
  default     = 1
}

variable "node_pool_tags" {
  description = "节点池的标签"
  type        = map(string)
  default     = {}
}

variable "root_volume_type" {
  description = "根卷的类型"
  type        = string
  default     = "SATA"
}

variable "root_volume_size" {
  description = "根卷的大小"
  type        = number
  default     = 40
}

variable "data_volumes_configuration" {
  description = "数据卷的配置"
  type = list(object({
    volumetype     = string
    size           = number
    count          = number
    kms_key_id     = optional(string, null)
    extend_params  = optional(map(string), null)
    virtual_spaces = optional(list(object({
      name            = string
      size            = string
      lvm_lv_type     = optional(string, null)
      lvm_path        = optional(string, null)
      runtime_lv_type = optional(string, null)
    })), [])
  }))

  default = [
    {
      volumetype = "SSD"
      size       = 100
      count      = 1
    }
  ]

  validation {
    condition     = length(var.data_volumes_configuration) > 0
    error_message = "至少需要提供一个数据卷"
  }
}

locals {
  flattened_data_volumes                                 = flatten([for v in var.data_volumes_configuration : [for i in range(v.count) : {
    volumetype    = v.volumetype
    size          = v.size
    kms_key_id    = v.kms_key_id
    extend_params = v.extend_params
  }]])
  default_data_volumes_configuration_with_virtual_spaces = [for v in slice(var.data_volumes_configuration, 0, 1) : v if length(v.virtual_spaces) > 0]
  user_data_volumes_configuration_with_virtual_spaces    = [for i, v in  [for v in slice(var.data_volumes_configuration, 1, length(var.data_volumes_configuration)) : v if length(v.virtual_spaces) > 0] : {
    select_name    = "user${i+1}"
    volumetype     = v.volumetype
    size           = v.size
    count          = v.count
    kms_key_id     = v.kms_key_id
    extend_params  = v.extend_params
    virtual_spaces = v.virtual_spaces
  }]
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE节点池资源，用于管理集群中的节点
resource "huaweicloud_cce_node_pool" "test" {
  cluster_id               = huaweicloud_cce_cluster.test.id
  type                     = var.node_pool_type
  name                     = var.node_pool_name
  flavor_id                = try(data.huaweicloud_compute_flavors.test.flavors[0].id, null)
  availability_zone        = try(data.huaweicloud_availability_zones.test.names[0], null)
  os                       = var.node_pool_os_type
  initial_node_count       = var.node_pool_initial_node_count
  min_node_count           = var.node_pool_min_node_count
  max_node_count           = var.node_pool_max_node_count
  scale_down_cooldown_time = var.node_pool_scale_down_cooldown_time
  priority                 = var.node_pool_priority
  key_pair                 = huaweicloud_kps_keypair.test.name
  tags                     = var.node_pool_tags

  root_volume {
    volumetype = var.root_volume_type
    size       = var.root_volume_size
  }

  dynamic "data_volumes" {
    for_each = local.flattened_data_volumes

    content {
      volumetype    = data_volumes.value.volumetype
      size          = data_volumes.value.size
      kms_key_id    = data_volumes.value.kms_key_id
      extend_params = data_volumes.value.extend_params
    }
  }

  dynamic "storage" {
    for_each = length(local.default_data_volumes_configuration_with_virtual_spaces) + length(local.user_data_volumes_configuration_with_virtual_spaces) > 0 ? [1] : []

    content {
      # 默认选择器，用于选择CCE使用的数据卷
      dynamic "selectors" {
        for_each = local.default_data_volumes_configuration_with_virtual_spaces

        content {
          name                           = "cceUse"
          type                           = "evs"
          match_label_volume_type        = selectors.value.volumetype
          match_label_size               = selectors.value.size
          match_label_count              = selectors.value.count
          match_label_metadata_encrypted = selectors.value.kms_key_id != "" && selectors.value.kms_key_id != null ? "1" : "0"
          match_label_metadata_cmkid     = selectors.value.kms_key_id != "" ? selectors.value.kms_key_id : null
        }
      }

      # 用户数据选择器，用于选择用户使用的数据卷
      dynamic "selectors" {
        for_each = local.user_data_volumes_configuration_with_virtual_spaces

        content {
          name                           = selectors.value.select_name
          type                           = "evs"
          match_label_volume_type        = selectors.value.volumetype
          match_label_size               = selectors.value.size
          match_label_count              = selectors.value.count
          match_label_metadata_encrypted = selectors.value.kms_key_id != "" && selectors.value.kms_key_id != null ? "1" : "0"
          match_label_metadata_cmkid     = selectors.value.kms_key_id != "" ? selectors.value.kms_key_id : null
        }
      }

      # 默认组，用于分组CCE使用的数据卷
      dynamic "groups" {
        for_each = local.default_data_volumes_configuration_with_virtual_spaces

        content {
          name           = "vgpaas"
          cce_managed    = true
          selector_names = ["cceUse"]

          dynamic "virtual_spaces" {
            for_each = groups.value.virtual_spaces

            content {
              name            = virtual_spaces.value.name
              size            = virtual_spaces.value.size
              lvm_lv_type     = virtual_spaces.value.lvm_lv_type
              lvm_path        = virtual_spaces.value.lvm_path
              runtime_lv_type = virtual_spaces.value.runtime_lv_type
            }
          }
        }
      }

      # 用户数据组，用于分组用户使用的数据卷
      dynamic "groups" {
        for_each = local.user_data_volumes_configuration_with_virtual_spaces

        content {
          name           = "vg${groups.value.select_name}"
          selector_names = [groups.value.select_name]

          dynamic "virtual_spaces" {
            for_each = groups.value.virtual_spaces

            content {
              name            = virtual_spaces.value.name
              size            = virtual_spaces.value.size
              lvm_lv_type     = virtual_spaces.value.lvm_lv_type
              lvm_path        = virtual_spaces.value.lvm_path
              runtime_lv_type = virtual_spaces.value.runtime_lv_type
            }
          }
        }
      }
    }
  }

  lifecycle {
    ignore_changes = [
      flavor_id,
      availability_zone,
      initial_node_count
    ]
  }

  depends_on = [
    huaweicloud_kps_keypair.test
  ]
}
```

**参数说明**：
- **cluster_id**：CCE集群ID，引用CCE集群资源（huaweicloud_cce_cluster.test）的ID进行赋值
- **type**：节点池的类型，通过引用输入变量 `node_pool_type` 进行赋值，默认为"vm"表示虚拟机类型
- **name**：节点池的名称，通过引用输入变量 `node_pool_name` 进行赋值
- **flavor_id**：节点规格ID，使用实例规格列表查询数据源的第一个规格ID进行赋值
- **availability_zone**：节点所在的可用区，使用可用区列表查询数据源的第一个可用区进行赋值
- **os**：节点的操作系统类型，通过引用输入变量 `node_pool_os_type` 进行赋值，默认为"EulerOS 2.9"
- **initial_node_count**：初始节点数量，通过引用输入变量 `node_pool_initial_node_count` 进行赋值，默认为2
- **min_node_count**：最小节点数量，通过引用输入变量 `node_pool_min_node_count` 进行赋值，默认为2
- **max_node_count**：最大节点数量，通过引用输入变量 `node_pool_max_node_count` 进行赋值，默认为10
- **scale_down_cooldown_time**：缩容冷却时间（分钟），通过引用输入变量 `node_pool_scale_down_cooldown_time` 进行赋值，默认为10分钟
- **priority**：节点池的优先级，通过引用输入变量 `node_pool_priority` 进行赋值，默认为1，数值越大优先级越高
- **key_pair**：密钥对名称，引用密钥对资源（huaweicloud_kps_keypair.test）的名称进行赋值
- **tags**：节点池的标签，通过引用输入变量 `node_pool_tags` 进行赋值，用于资源分类和管理
- **root_volume**：根卷配置块
  - **volumetype**：根卷类型，通过引用输入变量 `root_volume_type` 进行赋值，默认为"SATA"
  - **size**：根卷大小（GB），通过引用输入变量 `root_volume_size` 进行赋值，默认为40GB
- **data_volumes**：数据卷配置块，通过动态块（dynamic block）根据本地变量 `flattened_data_volumes` 创建多个数据卷配置
  - **volumetype**：数据卷类型，通过本地变量 `flattened_data_volumes` 中的 `volumetype` 进行赋值
  - **size**：数据卷大小（GB），通过本地变量 `flattened_data_volumes` 中的 `size` 进行赋值
  - **kms_key_id**：KMS密钥ID，通过本地变量 `flattened_data_volumes` 中的 `kms_key_id` 进行赋值，用于数据卷加密
  - **extend_params**：扩展参数，通过本地变量 `flattened_data_volumes` 中的 `extend_params` 进行赋值
- **storage**：存储配置块，用于配置数据卷的选择器、组和虚拟空间，仅当数据卷配置中包含虚拟空间时创建
  - **selectors**：选择器配置块，用于选择符合条件的数据卷
    - **name**：选择器名称，对于CCE使用的数据卷为"cceUse"，对于用户数据卷为"user1"、"user2"等
    - **type**：选择器类型，设置为"evs"表示云硬盘
    - **match_label_volume_type**：匹配标签-卷类型，通过数据卷配置中的 `volumetype` 进行赋值
    - **match_label_size**：匹配标签-大小，通过数据卷配置中的 `size` 进行赋值
    - **match_label_count**：匹配标签-数量，通过数据卷配置中的 `count` 进行赋值
    - **match_label_metadata_encrypted**：匹配标签-加密标识，如果配置了KMS密钥则为"1"，否则为"0"
    - **match_label_metadata_cmkid**：匹配标签-KMS密钥ID，如果配置了KMS密钥则使用该值，否则为null
  - **groups**：组配置块，用于分组数据卷并配置虚拟空间
    - **name**：组名称，对于CCE使用的数据卷为"vgpaas"，对于用户数据卷为"vguser1"、"vguser2"等
    - **cce_managed**：是否由CCE管理，对于CCE使用的数据卷为true，对于用户数据卷为false
    - **selector_names**：选择器名称列表，引用对应的选择器名称
    - **virtual_spaces**：虚拟空间配置块，通过动态块根据数据卷配置中的 `virtual_spaces` 创建虚拟空间
      - **name**：虚拟空间名称，例如"kubernetes"、"runtime"、"user"等
      - **size**：虚拟空间大小，可以是百分比（如"10%"）或固定大小
      - **lvm_lv_type**：LVM逻辑卷类型，例如"linear"表示线性卷
      - **lvm_path**：LVM路径，用于指定挂载路径，例如"/workspace"
      - **runtime_lv_type**：运行时逻辑卷类型

> 注意：`lifecycle` 块中的 `ignore_changes` 用于忽略某些字段的变更，避免在节点池创建后因这些字段的变更导致不必要的资源重建。`depends_on` 用于确保密钥对资源在节点池之前创建完成。

### 10. 预设资源部署所需的入参

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name              = "tf_test_vpc"
subnet_name           = "tf_test_subnet"
bandwidth_name        = "tf_test_bandwidth"
bandwidth_size        = 5
cluster_name          = "tf-test-cluster"
node_performance_type = "computingv3"
keypair_name          = "tf_test_keypair"
node_pool_name        = "tf-test-node-pool"
node_pool_tags        = {
  "owner" = "terraform"
}

root_volume_size           = 40
root_volume_type           = "SSD"
data_volumes_configuration = [
  {
    volumetype     = "SSD"
    size           = 100
    count          = 2
    virtual_spaces = [
      {
        name        = "kubernetes"
        size        = "10%"
        lvm_lv_type = "linear"
      },
      {
        name = "runtime"
        size = "90%"
      }
    ]
  },
  {
    volumetype     = "SSD"
    size           = 100
    count          = 1
    virtual_spaces = [
      {
        name        = "user"
        size        = "100%"
        lvm_lv_type = "linear"
        lvm_path    = "/workspace"
      }
    ]
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 11. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建节点池
4. 运行 `terraform show` 查看已创建的节点池

## 参考信息

- [华为云CCE产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE节点池最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cce/node-pool)
