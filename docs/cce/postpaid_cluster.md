# 使用Terraform部署按需计费CCE集群

## 概述

华为云云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。本最佳实践将介绍如何使用Terraform自动化部署按需计费的CCE集群。

### 应用场景

- 企业需要快速部署和管理容器集群
- 需要自动化部署和管理Kubernetes环境
- 需要统一管理多个容器集群
- 需要实现容器应用的弹性伸缩
- 需要按需使用和计费的弹性容器解决方案
- 临时项目或短期容器化需求

### 方案优势

- 自动化部署：使用Terraform实现基础设施即代码
- 按需付费：根据实际使用量计费，降低成本
- 弹性伸缩：支持节点和容器的自动扩缩容
- 安全可靠：提供多维度安全防护，支持网络隔离
- 快速交付：快速创建和配置容器环境
- 统一管理：集中管理容器资源和配置

### 涉及产品

- 云容器引擎（CCE）：提供容器集群管理服务
- 虚拟私有云（VPC）：提供隔离的网络环境
- 弹性云服务器（ECS）：作为集群节点
- 统一身份认证服务（IAM）：提供身份认证和权限管理

## 资源/数据源设计

本最佳实践涉及以下主要资源和数据源：

### 数据源

1. **可用区（data.huaweicloud_availability_zones）**
   - 用途：获取可用的可用区信息，用于创建CCE节点和ECS实例

2. **镜像（data.huaweicloud_images_image）**
   - 用途：获取ECS实例使用的操作系统镜像

### 资源

1. **VPC网络（huaweicloud_vpc）**
   - 用途：为CCE集群提供网络环境

2. **VPC子网（huaweicloud_vpc_subnet）**
   - 用途：在VPC中划分子网空间

3. **弹性公网IP（huaweicloud_vpc_eip）**
   - 用途：为CCE集群提供公网访问能力

4. **密钥对（huaweicloud_compute_keypair）**
   - 用途：为节点提供SSH密钥认证

5. **CCE集群（huaweicloud_cce_cluster）**
   - 用途：管理容器化应用

6. **CCE节点（huaweicloud_cce_node）**
   - 用途：作为CCE集群的工作节点

7. **计算实例（huaweicloud_compute_instance）**
   - 用途：创建ECS实例作为CCE集群的工作节点

8. **CCE节点挂载（huaweicloud_cce_node_attach）**
   - 用途：将已有ECS实例挂载到CCE集群

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.myaz
    ├── huaweicloud_cce_node.mynode
    └── huaweicloud_compute_instance.myecs

data.huaweicloud_images_image.myimage
    └── huaweicloud_compute_instance.myecs

huaweicloud_vpc.myvpc
    └── huaweicloud_vpc_subnet.mysubnet
        ├── huaweicloud_cce_cluster.mycce
        │   ├── huaweicloud_cce_node.mynode
        │   └── huaweicloud_cce_node_attach.test
        └── huaweicloud_compute_instance.myecs

huaweicloud_vpc_eip.myeip
    └── huaweicloud_cce_cluster.mycce

huaweicloud_compute_keypair.mykeypair
    ├── huaweicloud_cce_node.mynode
    ├── huaweicloud_compute_instance.myecs
    └── huaweicloud_cce_node_attach.test

huaweicloud_compute_instance.myecs
    └── huaweicloud_cce_node_attach.test
```

## 详细配置

### 数据源配置

#### 1. 可用区（data.huaweicloud_availability_zones）

获取基于当前provider块中所指定region下的所有可用区信息，用于创建CCE节点和ECS实例。

```hcl
data "huaweicloud_availability_zones" "myaz" {}
```

**参数说明**：
无需参数配置，默认获取当前区域下所有可用区信息。

#### 2. 镜像（data.huaweicloud_images_image）

获取ECS实例使用的操作系统镜像信息。

```hcl
variable "image_name" {
  description = "ECS镜像名称"
  type        = string
}

data "huaweicloud_images_image" "myimage" {
  name        = var.image_name
  most_recent = true
}
```

**参数说明**：
- **name**：镜像名称
- **most_recent**：是否使用最新版本的镜像

### 资源配置

#### 1. VPC网络（huaweicloud_vpc）

创建VPC网络环境，为CCE集群提供网络隔离。

```hcl
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
}

resource "huaweicloud_vpc" "myvpc" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称
- **cidr**：VPC网段，格式为CIDR

#### 2. VPC子网（huaweicloud_vpc_subnet）

在VPC中创建子网，为CCE集群提供网络空间。

```hcl
variable "subnet_name" {
  description = "子网名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网的CIDR块"
  type        = string
}

variable "subnet_gateway" {
  description = "子网的网关地址"
  type        = string
}

variable "primary_dns" {
  description = "子网的主DNS服务器"
  type        = string
}

variable "secondary_dns" {
  description = "子网的备DNS服务器"
  type        = string
}

resource "huaweicloud_vpc_subnet" "mysubnet" {
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway

  # dns is required for cce node installing
  primary_dns   = var.primary_dns
  secondary_dns = var.secondary_dns
  vpc_id        = huaweicloud_vpc.myvpc.id
}
```

**参数说明**：
- **name**：子网名称
- **cidr**：子网网段，格式为CIDR
- **gateway_ip**：网关IP地址
- **primary_dns**：主DNS服务器地址
- **secondary_dns**：备DNS服务器地址
- **vpc_id**：VPC ID

#### 3. 弹性公网IP（huaweicloud_vpc_eip）

创建弹性公网IP，为CCE集群提供公网访问能力。

```hcl
variable "bandwidth_name" {
  description = "带宽名称"
  type        = string
}

resource "huaweicloud_vpc_eip" "myeip" {
  publicip {
    type = "5_bgp"
  }
  bandwidth {
    name        = var.bandwidth_name
    size        = 8
    share_type  = "PER"
    charge_mode = "traffic"
  }
}
```

**参数说明**：
- **type**：弹性公网IP类型
- **name**：带宽名称
- **size**：带宽大小（Mbit/s）
- **share_type**：带宽共享类型
- **charge_mode**：计费模式

#### 4. 密钥对（huaweicloud_compute_keypair）

创建密钥对，为节点提供SSH密钥认证。

```hcl
variable "key_pair_name" {
  description = "密钥对名称"
  type        = string
}

resource "huaweicloud_compute_keypair" "mykeypair" {
  name = var.key_pair_name
}
```

**参数说明**：
- **name**：密钥对名称

#### 5. CCE集群（huaweicloud_cce_cluster）

创建CCE集群，提供容器编排和管理能力。

```hcl
variable "cce_cluster_name" {
  description = "CCE集群名称"
  type        = string
}

variable "cce_cluster_flavor" {
  description = "CCE集群规格"
  type        = string
}

resource "huaweicloud_cce_cluster" "mycce" {
  name                   = var.cce_cluster_name
  flavor_id              = var.cce_cluster_flavor
  vpc_id                 = huaweicloud_vpc.myvpc.id
  subnet_id              = huaweicloud_vpc_subnet.mysubnet.id
  container_network_type = "overlay_l2"
  eip                    = huaweicloud_vpc_eip.myeip.address
}
```

**参数说明**：
- **name**：集群名称
- **flavor_id**：集群规格
- **vpc_id**：VPC ID
- **subnet_id**：子网ID
- **container_network_type**：容器网络类型
- **eip**：弹性公网IP地址

#### 6. CCE节点（huaweicloud_cce_node）

创建CCE节点，作为集群的工作节点。

```hcl
variable "node_name" {
  description = "CCE节点名称"
  type        = string
}

variable "node_flavor" {
  description = "CCE节点规格"
  type        = string
}

variable "root_volume_size" {
  description = "系统盘大小"
  type        = number
}

variable "root_volume_type" {
  description = "系统盘类型"
  type        = string
}

variable "data_volume_size" {
  description = "数据盘大小"
  type        = number
}

variable "data_volume_type" {
  description = "数据盘类型"
  type        = string
}

resource "huaweicloud_cce_node" "mynode" {
  cluster_id        = huaweicloud_cce_cluster.mycce.id
  name              = var.node_name
  flavor_id         = var.node_flavor
  availability_zone = data.huaweicloud_availability_zones.myaz.names[0]
  key_pair          = huaweicloud_compute_keypair.mykeypair.name

  root_volume {
    size       = var.root_volume_size
    volumetype = var.root_volume_type
  }
  data_volumes {
    size       = var.data_volume_size
    volumetype = var.data_volume_type
  }
}
```

**参数说明**：
- **cluster_id**：CCE集群ID
- **name**：节点名称
- **flavor_id**：节点规格
- **availability_zone**：可用区
- **key_pair**：密钥对名称
- **root_volume.size**：系统盘大小
- **root_volume.volumetype**：系统盘类型
- **data_volumes.size**：数据盘大小
- **data_volumes.volumetype**：数据盘类型

#### 7. 计算实例（huaweicloud_compute_instance）

创建ECS实例，作为CCE集群的工作节点。

```hcl
variable "ecs_name" {
  description = "ECS实例名称"
  type        = string
}

variable "ecs_flavor" {
  description = "ECS实例规格"
  type        = string
}

resource "huaweicloud_compute_instance" "myecs" {
  name                        = var.ecs_name
  image_id                    = data.huaweicloud_images_image.myimage.id
  flavor_id                   = var.ecs_flavor
  availability_zone           = data.huaweicloud_availability_zones.myaz.names[0]
  key_pair                    = huaweicloud_compute_keypair.mykeypair.name
  delete_disks_on_termination = true

  system_disk_type = var.root_volume_type
  system_disk_size = var.root_volume_size

  data_disks {
    type = var.data_volume_type
    size = var.data_volume_size
  }

  network {
    uuid = huaweicloud_vpc_subnet.mysubnet.id
  }
}
```

**参数说明**：
- **name**：实例名称
- **image_id**：镜像ID
- **flavor_id**：实例规格
- **availability_zone**：可用区
- **key_pair**：密钥对名称
- **delete_disks_on_termination**：实例删除时是否删除磁盘
- **system_disk_type**：系统盘类型
- **system_disk_size**：系统盘大小
- **data_disks.type**：数据盘类型
- **data_disks.size**：数据盘大小
- **network.uuid**：网络ID

#### 8. CCE节点挂载（huaweicloud_cce_node_attach）

将ECS实例挂载到CCE集群作为工作节点。

```hcl
variable "os" {
  description = "操作系统类型"
  type        = string
}

resource "huaweicloud_cce_node_attach" "test" {
  cluster_id = huaweicloud_cce_cluster.mycce.id
  server_id  = huaweicloud_compute_instance.myecs.id
  key_pair   = huaweicloud_compute_keypair.mykeypair.name
  os         = var.os
}
```

**参数说明**：
- **cluster_id**：CCE集群ID
- **server_id**：ECS实例ID
- **key_pair**：密钥对名称
- **os**：操作系统类型

## 部署流程

1. 创建VPC和子网
2. 配置安全组和密钥对
3. 创建CCE集群
4. 部署工作节点

## 操作步骤

1. **准备工作**
   - 安装Terraform
   - 配置华为云认证信息
   - 创建工作目录

2. **创建Terraform配置文件**
   ```bash
   touch main.tf
   touch variables.tf
   ```

3. **初始化和部署**
   ```bash
   terraform init
   terraform plan
   terraform apply
   ```

4. **验证部署**
   - 登录CCE控制台
   - 检查集群和节点状态
   - 部署测试应用
   - 验证应用访问

## 注意事项

1. **网络规划**
   - 确保VPC网段和容器网段不重叠
   - 合理规划子网地址空间，预留扩展空间

2. **安全配置**
   - 启用RBAC认证
   - 配置合适的安全组规则
   - 妥善保管集群证书和密钥

3. **高可用设计**
   - 节点跨可用区部署
   - 合理配置节点的自动恢复策略
   - 关键应用配置多副本

4. **成本优化**
   - 选择合适的节点规格
   - 及时清理不使用的资源
   - 合理使用按需计费模式

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 自动化部署的CCE集群环境
2. 按需计费的资源使用模式
3. 安全可控的容器运行环境
4. 可重复使用的Terraform配置
5. 标准化的容器部署流程

## 参考信息

- [CCE产品文档](https://support.huaweicloud.com/intl/zh-cn/cce/)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE最佳实践](https://github.com/huaweicloud/terraform-provider-huaweicloud/blob/master/examples/cce/basic)
