# 部署节点分区

## 应用场景

云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。节点分区是CCE提供的一种资源隔离机制，用于将集群中的节点分配到不同的分区，实现资源的物理隔离和逻辑隔离。通过创建节点分区，可以将节点部署到指定的边缘站点或区域，满足边缘计算、混合云等场景的需求。本最佳实践将介绍如何使用Terraform自动化部署一个CCE节点分区，包括可用区、实例规格的查询，以及VPC、子网、ENI子网、CCE集群、节点分区、节点和节点池的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [计算规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCE集群资源（huaweicloud_cce_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_cluster)
- [CCE节点分区资源（huaweicloud_cce_partition）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_partition)
- [CCE节点资源（huaweicloud_cce_node）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_node)
- [CCE节点池资源（huaweicloud_cce_node_pool）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_node_pool)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_vpc_subnet
    ├── huaweicloud_vpc_subnet (ENI子网)
    └── data.huaweicloud_compute_flavors
        ├── huaweicloud_cce_node
        └── huaweicloud_cce_node_pool

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    │   └── huaweicloud_cce_cluster
    └── huaweicloud_vpc_subnet (ENI子网)
        └── huaweicloud_cce_partition
            ├── huaweicloud_cce_node
            └── huaweicloud_cce_node_pool

huaweicloud_cce_cluster
    └── huaweicloud_cce_partition
        ├── huaweicloud_cce_node
        └── huaweicloud_cce_node_pool
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询节点分区资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建节点分区相关资源：

```hcl
variable "availability_zone" {
  description = "创建CCE集群的可用区"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建节点分区相关资源
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询可用区信息，仅当 `var.availability_zone` 为空时查询可用区信息

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于为节点分区提供网络环境
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于为节点分区提供网络环境
resource "huaweicloud_vpc_subnet" "test" {
  count = var.subnet_id == "" ? 1 : 0

  vpc_id            = var.vpc_id != "" ? var.vpc_id : huaweicloud_vpc.test[0].id
  name              = var.subnet_name
  cidr              = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test[0].cidr, 4, 0)
  gateway_ip        = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : var.subnet_cidr != "" ? cidrhost(var.subnet_cidr, 1) : cidrhost(cidrsubnet(huaweicloud_vpc.test[0].cidr, 4, 0), 1)
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建VPC子网资源，仅当 `var.subnet_id` 为空时创建VPC子网资源
- **vpc_id**：子网所属的VPC ID，如果指定了VPC ID则使用该值，否则引用VPC资源（huaweicloud_vpc.test[0]）的ID进行赋值
- **name**：子网的名称，通过引用输入变量 `subnet_name` 进行赋值
- **cidr**：子网的CIDR块，如果指定了子网CIDR则使用该值，否则基于VPC的CIDR块通过 `cidrsubnet` 函数自动计算
- **gateway_ip**：子网的网关IP，如果指定了网关IP则使用该值，否则根据子网CIDR或自动计算的子网CIDR通过 `cidrhost` 函数自动计算
- **availability_zone**：子网所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区

### 5. 创建ENI子网资源（可选）

在TF文件中添加以下脚本以告知Terraform创建ENI子网资源（如果未指定ENI子网ID）：

```hcl
variable "eni_ipv4_subnet_id" {
  description = "ENI子网的ID"
  type        = string
  default     = ""
}

variable "eni_subnet_name" {
  description = "ENI子网的名称"
  type        = string
  default     = ""

  validation {
    condition     = var.eni_ipv4_subnet_id == "" || var.eni_subnet_name == ""
    error_message = "如果未提供eni_ipv4_subnet_id，则必须提供eni_subnet_name"
  }
}

variable "eni_subnet_cidr" {
  description = "ENI子网的CIDR块"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ENI子网资源，用于为节点分区提供ENI网络环境
resource "huaweicloud_vpc_subnet" "eni" {
  count = var.eni_ipv4_subnet_id == "" ? 1 : 0

  vpc_id            = var.vpc_id != "" ? var.vpc_id : huaweicloud_vpc.test[0].id
  name              = var.eni_subnet_name
  cidr              = var.eni_subnet_cidr != "" ? var.eni_subnet_cidr : cidrsubnet(huaweicloud_vpc.test[0].cidr, 4, 1)
  gateway_ip        = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : var.eni_subnet_cidr != "" ? cidrhost(var.eni_subnet_cidr, 1) : cidrhost(cidrsubnet(huaweicloud_vpc.test[0].cidr, 4, 1), 1)
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建ENI子网资源，仅当 `var.eni_ipv4_subnet_id` 为空时创建ENI子网资源
- **vpc_id**：ENI子网所属的VPC ID，如果指定了VPC ID则使用该值，否则引用VPC资源（huaweicloud_vpc.test[0]）的ID进行赋值
- **name**：ENI子网的名称，通过引用输入变量 `eni_subnet_name` 进行赋值
- **cidr**：ENI子网的CIDR块，如果指定了ENI子网CIDR则使用该值，否则基于VPC的CIDR块通过 `cidrsubnet` 函数自动计算（使用不同的子网索引以避免与普通子网冲突）
- **gateway_ip**：ENI子网的网关IP，如果指定了网关IP则使用该值，否则根据ENI子网CIDR或自动计算的ENI子网CIDR通过 `cidrhost` 函数自动计算
- **availability_zone**：ENI子网所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区

> 注意：ENI子网是节点分区必需的网络配置，用于提供高性能的网络连接。ENI子网必须与普通子网位于同一个VPC中，但需要使用不同的CIDR块以避免IP地址冲突。

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
  default     = "eni"
}

variable "cluster_description" {
  description = "CCE集群的描述"
  type        = string
  default     = ""
}

variable "cluster_tags" {
  description = "CCE集群的标签"
  type        = map(string)
  default     = {}
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE集群资源，用于部署和管理容器化应用
resource "huaweicloud_cce_cluster" "test" {
  name                         = var.cluster_name
  flavor_id                    = var.cluster_flavor_id
  cluster_version              = var.cluster_version
  cluster_type                 = var.cluster_type
  container_network_type       = var.container_network_type
  vpc_id                       = var.vpc_id != "" ? var.vpc_id : huaweicloud_vpc.test[0].id
  enable_distribute_management = true
  subnet_id                    = var.subnet_id != "" ? var.subnet_id : huaweicloud_vpc_subnet.test[0].id
  eni_subnet_id                = var.eni_ipv4_subnet_id != "" ? var.eni_ipv4_subnet_id : huaweicloud_vpc_subnet.eni[0].ipv4_subnet_id
  description                  = var.cluster_description
  tags                         = var.cluster_tags
}
```

**参数说明**：
- **name**：CCE集群的名称，通过引用输入变量 `cluster_name` 进行赋值
- **flavor_id**：CCE集群的规格ID，通过引用输入变量 `cluster_flavor_id` 进行赋值，默认为"cce.s1.small"表示小规格集群
- **cluster_version**：CCE集群的版本，通过引用输入变量 `cluster_version` 进行赋值，如果为null则使用最新版本
- **cluster_type**：CCE集群的类型，通过引用输入变量 `cluster_type` 进行赋值，默认为"VirtualMachine"表示虚拟机类型
- **container_network_type**：容器网络类型，通过引用输入变量 `container_network_type` 进行赋值，默认为"eni"表示ENI网络模式（节点分区必须使用ENI网络）
- **vpc_id**：VPC ID，如果指定了VPC ID则使用该值，否则引用VPC资源（huaweicloud_vpc.test[0]）的ID进行赋值
- **enable_distribute_management**：是否启用分布式管理，设置为true，这是节点分区功能的前置条件
- **subnet_id**：子网ID，如果指定了子网ID则使用该值，否则引用VPC子网资源（huaweicloud_vpc_subnet.test[0]）的ID进行赋值
- **eni_subnet_id**：ENI子网ID，如果指定了ENI子网ID则使用该值，否则引用ENI子网资源（huaweicloud_vpc_subnet.eni[0]）的IPv4子网ID进行赋值
- **description**：集群的描述信息，通过引用输入变量 `cluster_description` 进行赋值
- **tags**：集群的标签，通过引用输入变量 `cluster_tags` 进行赋值，用于资源分类和管理

> 注意：节点分区功能要求集群必须启用分布式管理（`enable_distribute_management`设置为true），并且必须使用ENI网络模式（`container_network_type`设置为"eni"）。

### 7. 通过数据源查询节点分区资源创建所需的实例规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的实例规格：

```hcl
variable "node_flavor_id" {
  description = "节点的规格ID"
  type        = string
  default     = ""
}

variable "node_flavor_performance_type" {
  description = "节点的性能类型"
  type        = string
  default     = "normal"
}

variable "node_flavor_cpu_core_count" {
  description = "节点的CPU核心数"
  type        = number
  default     = 2
}

variable "node_flavor_memory_size" {
  description = "节点的内存大小"
  type        = number
  default     = 4
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的实例规格信息，用于创建节点分区相关资源
data "huaweicloud_compute_flavors" "test" {
  count = var.node_flavor_id == "" ? 1 : 0

  performance_type  = var.node_flavor_performance_type
  cpu_core_count    = var.node_flavor_cpu_core_count
  memory_size       = var.node_flavor_memory_size
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询实例规格信息，仅当 `var.node_flavor_id` 为空时查询实例规格信息
- **performance_type**：性能类型，通过引用输入变量 `node_flavor_performance_type` 进行赋值，默认为"normal"表示通用型
- **cpu_core_count**：CPU核心数，通过引用输入变量 `node_flavor_cpu_core_count` 进行赋值，默认为2核
- **memory_size**：内存大小（GB），通过引用输入变量 `node_flavor_memory_size` 进行赋值，默认为4GB
- **availability_zone**：实例规格所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区

### 8. 创建CCE节点分区资源

在TF文件中添加以下脚本以告知Terraform创建CCE节点分区资源：

```hcl
variable "node_partition" {
  description = "节点分区的名称或ID"
  type        = string
  default     = ""

  validation {
    condition     = var.node_partition != "" || var.partition_name != ""
    error_message = "如果未提供node_partition，则必须提供partition_name"
  }
}

variable "partition_name" {
  description = "节点分区的名称"
  type        = string
  default     = ""
}

variable "partition_category" {
  description = "节点分区的类别"
  type        = string
  default     = "IES"
}

variable "partition_public_border_group" {
  description = "节点分区的公共边界组"
  type        = string
  default     = ""

  validation {
    condition     = var.node_partition == "" || var.partition_public_border_group == ""
    error_message = "如果未提供node_partition，则必须提供partition_public_border_group"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE节点分区资源，用于将节点分配到指定的分区
resource "huaweicloud_cce_partition" "test" {
  count = var.node_partition == "" ? 1 : 0

  cluster_id           = huaweicloud_cce_cluster.test.id
  name                 = var.partition_name
  category             = var.partition_category
  public_border_group  = var.partition_public_border_group
  partition_subnet_id  = huaweicloud_vpc_subnet.eni[0].id
  container_subnet_ids = [huaweicloud_vpc_subnet.eni[0].ipv4_subnet_id]
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建节点分区资源，仅当 `var.node_partition` 为空时创建节点分区资源
- **cluster_id**：CCE集群ID，引用CCE集群资源（huaweicloud_cce_cluster.test）的ID进行赋值
- **name**：节点分区的名称，通过引用输入变量 `partition_name` 进行赋值
- **category**：节点分区的类别，通过引用输入变量 `partition_category` 进行赋值，默认为"IES"表示智能边缘站点
- **public_border_group**：节点分区的公共边界组，通过引用输入变量 `partition_public_border_group` 进行赋值，用于指定边缘站点的位置
- **partition_subnet_id**：分区子网ID，引用ENI子网资源（huaweicloud_vpc_subnet.eni[0]）的ID进行赋值
- **container_subnet_ids**：容器子网ID列表，引用ENI子网资源（huaweicloud_vpc_subnet.eni[0]）的IPv4子网ID列表进行赋值

> 注意：节点分区用于将节点分配到指定的边缘站点或区域。如果已经存在节点分区，可以通过 `node_partition` 变量指定分区ID，无需创建新的分区。

### 9. 创建CCE节点资源（可选）

在TF文件中添加以下脚本以告知Terraform创建CCE节点资源：

```hcl
variable "node_name" {
  description = "节点的名称"
  type        = string
}

variable "node_password" {
  description = "节点的root密码"
  type        = string
  sensitive   = true
}

variable "root_volume_type" {
  description = "根卷的类型"
  type        = string
  default     = "SSD"
}

variable "root_volume_size" {
  description = "根卷的大小"
  type        = number
  default     = 40
}

variable "data_volumes_configuration" {
  description = "数据卷的配置"
  type = list(object({
    volumetype = string
    size       = number
  }))

  default = []
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE节点资源，用于为集群提供计算能力
resource "huaweicloud_cce_node" "test" {
  cluster_id        = huaweicloud_cce_cluster.test.id
  name              = var.node_name
  flavor_id         = var.node_flavor_id != "" ? var.node_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  password          = var.node_password
  partition         = var.node_partition != "" ? var.node_partition : huaweicloud_cce_partition.test[0].id

  root_volume {
    volumetype = var.root_volume_type
    size       = var.root_volume_size
  }

  dynamic "data_volumes" {
    for_each = var.data_volumes_configuration

    content {
      volumetype = data_volumes.value.volumetype
      size       = data_volumes.value.size
    }
  }

  lifecycle {
    ignore_changes = [
      flavor_id,
      availability_zone,
    ]
  }
}
```

**参数说明**：
- **cluster_id**：CCE集群ID，引用CCE集群资源（huaweicloud_cce_cluster.test）的ID进行赋值
- **name**：节点的名称，通过引用输入变量 `node_name` 进行赋值
- **flavor_id**：节点规格ID，如果指定了节点规格ID则使用该值，否则使用实例规格列表查询数据源的第一个规格ID进行赋值
- **availability_zone**：节点所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区进行赋值
- **password**：节点的root密码，通过引用输入变量 `node_password` 进行赋值，用于SSH登录节点
- **partition**：节点所属的分区，如果指定了节点分区ID则使用该值，否则引用节点分区资源（huaweicloud_cce_partition.test[0]）的ID进行赋值
- **root_volume**：根卷配置块
  - **volumetype**：根卷类型，通过引用输入变量 `root_volume_type` 进行赋值，默认为"SSD"
  - **size**：根卷大小（GB），通过引用输入变量 `root_volume_size` 进行赋值，默认为40GB
- **data_volumes**：数据卷配置块，通过动态块（dynamic block）根据输入变量 `data_volumes_configuration` 创建多个数据卷配置
  - **volumetype**：数据卷类型，通过输入变量 `data_volumes_configuration` 中的 `volumetype` 进行赋值
  - **size**：数据卷大小（GB），通过输入变量 `data_volumes_configuration` 中的 `size` 进行赋值
- **lifecycle**：生命周期配置块，用于忽略某些字段的变更，避免在节点创建后因这些字段的变更导致不必要的资源重建

> 注意：节点必须指定所属的分区，通过 `partition` 参数进行配置。节点分区用于将节点部署到指定的边缘站点或区域。

### 10. 创建CCE节点池资源（可选）

在TF文件中添加以下脚本以告知Terraform创建CCE节点池资源：

```hcl
variable "node_pool_name" {
  description = "节点池的名称"
  type        = string
  default     = ""
}

variable "node_pool_os_type" {
  description = "节点池的操作系统类型"
  type        = string
  default     = "EulerOS 2.9"
}

variable "node_pool_initial_node_count" {
  description = "节点池的初始节点数量"
  type        = number
  default     = 1
}

variable "node_pool_password" {
  description = "节点池的root密码"
  type        = string
  sensitive   = true
  default     = ""

  validation {
    condition     = var.node_pool_name != "" || var.node_pool_password != ""
    error_message = "如果未提供node_pool_name，则必须提供node_pool_password"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE节点池资源，用于管理集群中的节点
resource "huaweicloud_cce_node_pool" "test" {
  count = var.node_pool_name != "" ? 1 : 0

  cluster_id         = huaweicloud_cce_cluster.test.id
  name               = var.node_pool_name
  os                 = var.node_pool_os_type
  flavor_id          = var.node_flavor_id != "" ? var.node_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  initial_node_count = var.node_pool_initial_node_count
  availability_zone  = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  password           = var.node_pool_password
  type               = "vm"
  partition          = var.node_partition != "" ? var.node_partition : huaweicloud_cce_partition.test[0].id

  root_volume {
    volumetype = var.root_volume_type
    size       = var.root_volume_size
  }

  dynamic "data_volumes" {
    for_each = var.data_volumes_configuration

    content {
      volumetype = data_volumes.value.volumetype
      size       = data_volumes.value.size
    }
  }

  lifecycle {
    ignore_changes = [
      flavor_id,
      availability_zone,
    ]
  }
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建节点池资源，仅当 `var.node_pool_name` 不为空时创建节点池资源
- **cluster_id**：CCE集群ID，引用CCE集群资源（huaweicloud_cce_cluster.test）的ID进行赋值
- **name**：节点池的名称，通过引用输入变量 `node_pool_name` 进行赋值
- **os**：节点的操作系统类型，通过引用输入变量 `node_pool_os_type` 进行赋值，默认为"EulerOS 2.9"
- **flavor_id**：节点规格ID，如果指定了节点规格ID则使用该值，否则使用实例规格列表查询数据源的第一个规格ID进行赋值
- **initial_node_count**：初始节点数量，通过引用输入变量 `node_pool_initial_node_count` 进行赋值，默认为1
- **availability_zone**：节点所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区进行赋值
- **password**：节点的root密码，通过引用输入变量 `node_pool_password` 进行赋值，用于SSH登录节点
- **type**：节点池的类型，设置为"vm"表示虚拟机类型
- **partition**：节点池所属的分区，如果指定了节点分区ID则使用该值，否则引用节点分区资源（huaweicloud_cce_partition.test[0]）的ID进行赋值
- **root_volume**：根卷配置块
  - **volumetype**：根卷类型，通过引用输入变量 `root_volume_type` 进行赋值，默认为"SSD"
  - **size**：根卷大小（GB），通过引用输入变量 `root_volume_size` 进行赋值，默认为40GB
- **data_volumes**：数据卷配置块，通过动态块（dynamic block）根据输入变量 `data_volumes_configuration` 创建多个数据卷配置
  - **volumetype**：数据卷类型，通过输入变量 `data_volumes_configuration` 中的 `volumetype` 进行赋值
  - **size**：数据卷大小（GB），通过输入变量 `data_volumes_configuration` 中的 `size` 进行赋值
- **lifecycle**：生命周期配置块，用于忽略某些字段的变更，避免在节点池创建后因这些字段的变更导致不必要的资源重建

> 注意：节点池必须指定所属的分区，通过 `partition` 参数进行配置。节点池中的所有节点都会部署到指定的分区中。

### 11. 预设资源部署所需的入参

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name                     = "tf_test_vpc"
subnet_name                  = "tf_test_subnet"
eni_subnet_name              = "tf_test_eni_subnet"
cluster_name                 = "tf-test-cluster"
node_partition               = "center"
node_flavor_id               = "c7n.large.2"
node_flavor_performance_type = "computingv3"
node_name                    = "tf-test-node"
node_password                = "your_node_password"
data_volumes_configuration   = [
  {
    volumetype = "SSD"
    size       = 100
  }
]

node_pool_password = "your_node_pool_password"
node_pool_name     = "tf-test-node-pool"
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

### 12. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建节点分区
4. 运行 `terraform show` 查看已创建的节点分区

## 参考信息

- [华为云CCE产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE节点分区最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cce/node-partition)
