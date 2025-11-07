# 使用新OBS部署Kubernetes PVC

## 应用场景

云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。持久化卷声明（Persistent Volume Claim，PVC）是Kubernetes中用于请求存储资源的抽象接口，允许Pod通过声明的方式请求存储资源，而无需关心底层存储的具体实现。对象存储服务（Object Storage Service，OBS）是华为云提供的高可用、高可靠、高性能、安全、低成本的对象存储服务，可以作为Kubernetes集群的持久化存储后端。

通过将OBS桶作为Kubernetes的持久化存储，可以为容器应用提供可扩展、高可用的存储解决方案。这种方式特别适合需要共享存储、大容量存储或跨可用区数据复制的应用场景。与使用现有OBS桶的方式不同，本最佳实践通过PVC自动创建OBS桶和Persistent Volume，简化了部署流程。本最佳实践将介绍如何使用Terraform自动化部署一个通过新OBS管理PVC的完整方案，包括可用区、实例规格的查询，以及VPC、子网、弹性公网IP、CCE集群、节点等基础设施的创建，以及Kubernetes Secret、Persistent Volume Claim和Deployment的创建。

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
- [CCE节点资源（huaweicloud_cce_node）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_node)
- [Kubernetes Secret资源（kubernetes_secret）](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/secret)
- [Kubernetes持久化卷声明资源（kubernetes_persistent_volume_claim）](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/persistent_volume_claim)
- [Kubernetes Deployment资源（kubernetes_deployment）](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/deployment)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_vpc_subnet
    └── data.huaweicloud_compute_flavors
        └── huaweicloud_cce_node

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_cce_cluster
            └── huaweicloud_cce_node
                └── kubernetes_deployment

huaweicloud_vpc_eip
    └── huaweicloud_cce_cluster
        └── huaweicloud_cce_node
            └── kubernetes_deployment

huaweicloud_kps_keypair
    └── huaweicloud_cce_node
        └── kubernetes_deployment

kubernetes_secret
    └── kubernetes_persistent_volume_claim
        └── kubernetes_deployment
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 配置Kubernetes Provider

由于本最佳实践需要使用Kubernetes provider来创建Kubernetes资源，需要在providers.tf文件中配置Kubernetes provider。在providers.tf文件中添加以下脚本：

```hcl
terraform {
  required_version = ">= 1.9.0"

  required_providers {
    ...
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 1.6.2"
    }
  }
}

variable "eip_address" {
  description = "CCE集群的EIP地址"
  type        = string
  default     = ""
}

# 配置Kubernetes provider，用于创建Kubernetes资源
provider "kubernetes" {
  host                   = "https://%{ if var.eip_address != "" }${var.eip_address}:5443%{ else }${huaweicloud_vpc_eip.test[0].address}:5443%{ endif }"
  cluster_ca_certificate = base64decode(huaweicloud_cce_cluster.test.certificate_clusters[0].certificate_authority_data)
  client_certificate     = base64decode(huaweicloud_cce_cluster.test.certificate_users[0].client_certificate_data)
  client_key             = base64decode(huaweicloud_cce_cluster.test.certificate_users[0].client_key_data)
}
```

**参数说明**：
- **host**：Kubernetes API服务器的地址，如果指定了EIP地址则使用该值，否则引用弹性公网IP资源（huaweicloud_vpc_eip.test[0]）的地址，端口为5443
- **cluster_ca_certificate**：集群CA证书，从CCE集群资源（huaweicloud_cce_cluster.test）的证书信息中获取并解码
- **client_certificate**：客户端证书，从CCE集群资源（huaweicloud_cce_cluster.test）的证书信息中获取并解码
- **client_key**：客户端密钥，从CCE集群资源（huaweicloud_cce_cluster.test）的证书信息中获取并解码

> 注意：Kubernetes provider需要访问CCE集群的API服务器，因此需要配置正确的集群地址和证书信息。这些信息可以从CCE集群资源中获取。

### 3. 通过数据源查询PVC资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建PVC相关资源：

```hcl
variable "availability_zone" {
  description = "资源所在的可用区"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建PVC相关资源
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zone` 为空时创建数据源（即执行可用区列表查询）

### 4. 创建VPC资源（可选）

在TF文件中添加以下脚本以告知Terraform创建VPC资源（如果未指定VPC ID和子网ID）：

```hcl
variable "vpc_id" {
  description = "VPC的ID"
  type        = string
  default     = ""
  nullable    = false
}

variable "vpc_name" {
  description = "VPC的名称"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.vpc_id != "" || var.vpc_name != ""
    error_message = "如果未提供vpc_id，则必须提供vpc_name。"
  }
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于为PVC提供网络环境
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

### 5. 创建VPC子网资源（可选）

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源（如果未指定子网ID）：

```hcl
variable "subnet_id" {
  description = "子网的ID"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_name" {
  description = "子网的名称"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.subnet_id == "" || var.subnet_name == ""
    error_message = "如果未提供subnet_id，则必须提供subnet_name。"
  }
}

variable "subnet_cidr" {
  description = "子网的CIDR块"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "子网的网关IP"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于为PVC提供网络环境
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

### 6. 创建弹性公网IP资源（可选）

在TF文件中添加以下脚本以告知Terraform创建弹性公网IP资源（如果未指定EIP地址）：

```hcl
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源，用于为PVC提供公网访问能力
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

### 7. 创建CCE集群资源

在TF文件中添加以下脚本以告知Terraform创建CCE集群资源：

```hcl
variable "cluster_name" {
  description = "CCE集群的名称"
  type        = string
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

variable "authentication_mode" {
  description = "CCE集群的认证模式"
  type        = string
  default     = "rbac"
}

variable "delete_all_resources_on_terminal" {
  description = "是否在终止时删除所有资源"
  type        = bool
  default     = true
}

variable "enterprise_project_id" {
  description = "CCE集群的企业项目ID"
  type        = string
  default     = "0"
}

variable "eip_address" {
  description = "CCE集群的EIP地址"
  type        = string
  default     = ""
  nullable    = false
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
  authentication_mode    = var.authentication_mode
  delete_all             = var.delete_all_resources_on_terminal ? "true" : "false"
  enterprise_project_id  = var.enterprise_project_id
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
- **authentication_mode**：集群的认证模式，通过引用输入变量 `authentication_mode` 进行赋值，默认为"rbac"表示基于角色的访问控制
- **delete_all**：是否在终止时删除所有资源，通过引用输入变量 `delete_all_resources_on_terminal` 进行赋值，默认为"true"表示删除所有资源
- **enterprise_project_id**：企业项目ID，通过引用输入变量 `enterprise_project_id` 进行赋值，默认为"0"表示默认企业项目

### 8. 通过数据源查询节点资源创建所需的实例规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的实例规格：

```hcl
variable "node_flavor_id" {
  description = "节点的规格ID"
  type        = string
  default     = ""
  nullable    = false
}

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

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的实例规格信息，用于创建PVC相关资源
data "huaweicloud_compute_flavors" "test" {
  count = var.node_flavor_id == "" ? 1 : 0

  performance_type  = var.node_performance_type
  cpu_core_count    = var.node_cpu_core_count
  memory_size       = var.node_memory_size
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询实例规格信息，仅当 `var.node_flavor_id` 为空时查询实例规格信息
- **performance_type**：性能类型，通过引用输入变量 `node_performance_type` 进行赋值，默认为"general"表示通用型
- **cpu_core_count**：CPU核心数，通过引用输入变量 `node_cpu_core_count` 进行赋值，默认为4核
- **memory_size**：内存大小（GB），通过引用输入变量 `node_memory_size` 进行赋值，默认为8GB
- **availability_zone**：实例规格所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区

### 9. 创建密钥对资源

在TF文件中添加以下脚本以告知Terraform创建密钥对资源：

```hcl
variable "keypair_name" {
  description = "密钥对的名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建密钥对资源，用于访问节点
resource "huaweicloud_kps_keypair" "test" {
  name = var.keypair_name
}
```

**参数说明**：
- **name**：密钥对的名称，通过引用输入变量 `keypair_name` 进行赋值

### 10. 创建CCE节点资源

在TF文件中添加以下脚本以告知Terraform创建CCE节点资源：

```hcl
variable "node_name" {
  description = "节点的名称"
  type        = string
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
    volumetype = string
    size       = number
  }))

  default  = []
  nullable = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE节点资源，用于为集群提供计算能力
resource "huaweicloud_cce_node" "test" {
  cluster_id        = huaweicloud_cce_cluster.test.id
  name              = var.node_name
  flavor_id         = var.node_flavor_id != "" ? var.node_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  key_pair          = huaweicloud_kps_keypair.test.name

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
}
```

**参数说明**：
- **cluster_id**：节点所属的CCE集群ID，引用CCE集群资源（huaweicloud_cce_cluster.test）的ID进行赋值
- **name**：节点的名称，通过引用输入变量 `node_name` 进行赋值
- **flavor_id**：节点的规格ID，如果指定了节点规格ID则使用该值，否则根据计算规格列表查询数据源的返回结果进行赋值
- **availability_zone**：节点所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区
- **key_pair**：节点使用的密钥对名称，引用密钥对资源（huaweicloud_kps_keypair.test）的名称进行赋值
- **root_volume**：根卷配置块
  - **volumetype**：根卷类型，通过引用输入变量 `root_volume_type` 进行赋值，默认为"SATA"
  - **size**：根卷大小（GB），通过引用输入变量 `root_volume_size` 进行赋值，默认为40GB
- **data_volumes**：数据卷配置块（动态块），根据输入变量 `data_volumes_configuration` 动态创建
  - **volumetype**：数据卷类型，通过引用输入变量中的数据卷配置进行赋值
  - **size**：数据卷大小（GB），通过引用输入变量中的数据卷配置进行赋值

### 11. 创建Kubernetes Secret资源

在TF文件中添加以下脚本以告知Terraform创建Kubernetes Secret资源：

```hcl
variable "secret_name" {
  description = "Kubernetes Secret的名称"
  type        = string
}

variable "namespace_name" {
  description = "CCE命名空间的名称"
  type        = string
  default     = "default"
}

variable "secret_labels" {
  description = "Kubernetes Secret的标签"
  type        = map(string)
  default     = {
    "secret.kubernetes.io/used-by" = "csi"
  }
}

variable "secret_data" {
  description = "Kubernetes Secret的数据"
  type        = map(string)
  sensitive   = true
}

variable "secret_type" {
  description = "Kubernetes Secret的类型"
  type        = string
  default     = "cfe/secure-opaque"
}

# 创建Kubernetes Secret资源，用于存储OBS访问凭证
resource "kubernetes_secret" "test" {
  metadata {
    name      = var.secret_name
    namespace = var.namespace_name
    labels    = var.secret_labels
  }

  data = var.secret_data
  type = var.secret_type

  # 对于`data`参数，输入值与状态值不一致，因此需要忽略更改
  lifecycle {
    ignore_changes = [data]
  }
}
```

**参数说明**：
- **metadata**：元数据配置块
  - **name**：Secret的名称，通过引用输入变量 `secret_name` 进行赋值
  - **namespace**：Secret所在的命名空间，通过引用输入变量 `namespace_name` 进行赋值，默认为"default"
  - **labels**：Secret的标签，通过引用输入变量 `secret_labels` 进行赋值，默认为包含"secret.kubernetes.io/used-by"标签
- **data**：Secret的数据，通过引用输入变量 `secret_data` 进行赋值，包含OBS访问密钥和密钥
- **type**：Secret的类型，通过引用输入变量 `secret_type` 进行赋值，默认为"cfe/secure-opaque"
- **lifecycle**：生命周期配置块，用于忽略`data`参数的更改，因为Secret的数据可能会在外部被修改

### 12. 创建Kubernetes Persistent Volume Claim资源

在TF文件中添加以下脚本以告知Terraform创建Kubernetes Persistent Volume Claim资源：

```hcl
variable "pvc_name" {
  description = "持久化卷声明的名称"
  type        = string
}

variable "pvc_obs_volume_type" {
  description = "PVC的OBS卷类型"
  type        = string
  default     = "standard"
}

variable "pvc_fstype" {
  description = "PVC的文件系统类型"
  type        = string
  default     = "s3fs"
}

variable "pvc_access_modes" {
  description = "PVC的访问模式"
  type        = list(string)
  default     = ["ReadWriteMany"]
}

variable "pvc_storage" {
  description = "PVC的存储大小"
  type        = string
  default     = "1Gi"
}

variable "pvc_storage_class_name" {
  description = "PVC的存储类名称"
  type        = string
  default     = "csi-obs"
}

variable "enterprise_project_id" {
  description = "企业项目ID"
  type        = string
  default     = "0"
}

# 创建Kubernetes Persistent Volume Claim资源，用于请求存储资源，PVC会自动创建Persistent Volume和OBS桶
resource "kubernetes_persistent_volume_claim" "test" {
  metadata {
    name      = var.pvc_name
    namespace = var.namespace_name

    annotations = {
      "everest.io/obs-volume-type"                       = var.pvc_obs_volume_type
      "csi.storage.k8s.io/fstype"                        = var.pvc_fstype
      "csi.storage.k8s.io/node-publish-secret-name"      = kubernetes_secret.test.metadata[0].name
      "csi.storage.k8s.io/node-publish-secret-namespace" = var.namespace_name
      "everest.io/enterprise-project-id"                 = var.enterprise_project_id
    }
  }

  spec {
    access_modes = var.pvc_access_modes

    resources {
      requests = {
        storage = var.pvc_storage
      }
    }

    storage_class_name = var.pvc_storage_class_name
  }
}
```

**参数说明**：
- **metadata**：元数据配置块
  - **name**：Persistent Volume Claim的名称，通过引用输入变量 `pvc_name` 进行赋值
  - **namespace**：Persistent Volume Claim所在的命名空间，通过引用输入变量 `namespace_name` 进行赋值
  - **annotations**：注解，包含OBS卷类型、文件系统类型、Secret引用和企业项目ID
- **spec**：规格配置块
  - **access_modes**：访问模式列表，通过引用输入变量 `pvc_access_modes` 进行赋值，默认为["ReadWriteMany"]表示多节点读写
  - **resources**：资源请求配置块
    - **requests**：资源请求，包含存储大小请求
      - **storage**：存储大小，通过引用输入变量 `pvc_storage` 进行赋值，默认为"1Gi"
  - **storage_class_name**：存储类名称，通过引用输入变量 `pvc_storage_class_name` 进行赋值，默认为"csi-obs"，当使用该存储类时，Kubernetes会自动创建Persistent Volume和OBS桶

> 注意：与使用现有OBS桶的方式不同，本最佳实践通过PVC直接使用存储类（storage_class_name），Kubernetes会自动创建Persistent Volume和OBS桶，无需手动创建这些资源。

### 13. 创建Kubernetes Deployment资源

在TF文件中添加以下脚本以告知Terraform创建Kubernetes Deployment资源：

```hcl
variable "deployment_name" {
  description = "Deployment的名称"
  type        = string
}

variable "deployment_replicas" {
  description = "Deployment的Pod副本数"
  type        = number
  default     = 2
}

variable "deployment_containers" {
  description = "Deployment的容器列表"
  type = list(object({
    name          = string
    image         = string
    volume_mounts = list(object({
      mount_path = string
    }))
  }))
  nullable = false
}

variable "deployment_volume_name" {
  description = "Deployment的卷名称"
  type        = string
  default     = "pvc-obs-volume"
}

variable "deployment_image_pull_secrets" {
  description = "Deployment的镜像拉取密钥"
  type        = list(string)
  default     = ["default-secret"]
  nullable    = false
}

# 创建Kubernetes Deployment资源，用于部署使用PVC的应用
resource "kubernetes_deployment" "test" {
  metadata {
    name      = var.deployment_name
    namespace = var.namespace_name
  }

  spec {
    replicas = var.deployment_replicas

    selector {
      match_labels = {
        app = var.deployment_name
      }
    }

    template {
      metadata {
        labels = {
          app = var.deployment_name
        }
      }

      spec {
        dynamic "container" {
          for_each = var.deployment_containers

          content {
            name  = container.value.name
            image = container.value.image

            dynamic "volume_mount" {
              for_each = container.value.volume_mounts

              content {
                name       = var.deployment_volume_name
                mount_path = volume_mount.value.mount_path
              }
            }
          }
        }

        dynamic "image_pull_secrets" {
          for_each = var.deployment_image_pull_secrets

          content {
            name = image_pull_secrets.value
          }
        }

        volume {
          name = var.deployment_volume_name

          persistent_volume_claim {
            claim_name = kubernetes_persistent_volume_claim.test.metadata[0].name
          }
        }
      }
    }
  }

  depends_on = [huaweicloud_cce_node.test]
}
```

**参数说明**：
- **metadata**：元数据配置块
  - **name**：Deployment的名称，通过引用输入变量 `deployment_name` 进行赋值
  - **namespace**：Deployment所在的命名空间，通过引用输入变量 `namespace_name` 进行赋值
- **spec**：规格配置块
  - **replicas**：Pod副本数，通过引用输入变量 `deployment_replicas` 进行赋值，默认为2
  - **selector**：选择器配置块，用于选择Pod
    - **match_labels**：匹配标签，包含应用名称标签
  - **template**：Pod模板配置块
    - **metadata**：Pod元数据配置块
      - **labels**：Pod标签，包含应用名称标签
    - **spec**：Pod规格配置块
      - **container**：容器配置块（动态块），根据输入变量 `deployment_containers` 动态创建
        - **name**：容器名称，通过引用输入变量中的容器配置进行赋值
        - **image**：容器镜像，通过引用输入变量中的容器配置进行赋值
        - **volume_mount**：卷挂载配置块（动态块），根据输入变量中的容器卷挂载配置动态创建
          - **name**：卷名称，通过引用输入变量 `deployment_volume_name` 进行赋值，默认为"pvc-obs-volume"
          - **mount_path**：挂载路径，通过引用输入变量中的卷挂载配置进行赋值
      - **image_pull_secrets**：镜像拉取密钥配置块（动态块），根据输入变量 `deployment_image_pull_secrets` 动态创建
        - **name**：密钥名称，通过引用输入变量中的镜像拉取密钥字符串进行赋值
      - **volume**：卷配置块
        - **name**：卷名称，通过引用输入变量 `deployment_volume_name` 进行赋值
        - **persistent_volume_claim**：持久化卷声明配置块
          - **claim_name**：声明名称，引用Kubernetes Persistent Volume Claim资源（kubernetes_persistent_volume_claim.test）的名称进行赋值
- **depends_on**：显式依赖，确保CCE节点资源创建完成后再创建Deployment

### 14. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 网络资源配置
vpc_name       = "tf_test_vpc"
subnet_name    = "tf_test_subnet"
bandwidth_name = "tf_test_bandwidth"
bandwidth_size = 5

# CCE集群配置
cluster_name          = "tf-test-cluster"
node_performance_type  = "computingv3"
keypair_name          = "tf_test_keypair"
node_name             = "tf-test-node"
root_volume_size      = 40
root_volume_type      = "SSD"
data_volumes_configuration = [
  {
    volumetype = "SSD"
    size       = 100
  }
]

# OBS和Kubernetes资源配置
secret_name = "tf-test-secret"
secret_data = {
  "access.key" = "your_access_key"
  "secret.key" = "your_secret_key"
}

pvc_name              = "tf-test-pvc-obs"
deployment_name       = "tf-test-deployment"
deployment_containers = [
  {
    name          = "container-1"
    image         = "nginx:latest"
    volume_mounts = [
      {
        mount_path = "/data"
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

### 15. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建通过新OBS管理PVC的完整方案
4. 运行 `terraform show` 查看已创建的通过新OBS管理PVC的完整方案

## 参考信息

- [华为云CCE产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云OBS产品文档](https://support.huaweicloud.com/obs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Kubernetes Provider文档](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs)
- [CCE使用新OBS部署Kubernetes PVC最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cce/kubenetes/pvc-with-new-obs-bucket)
