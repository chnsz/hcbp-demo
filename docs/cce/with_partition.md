# 使用Terraform创建带有分区的CCE节点

## 概述

华为云云容器引擎（Cloud Container Engine，CCE）是一种高可用、高性能的企业级Kubernetes容器管理服务，支持多可用区容灾，提供资源池化能力。本最佳实践将介绍如何使用Terraform自动化部署带有分区的CCE集群，通过ENI网络模式和分区管理，实现工作负载在不同可用区间的均衡分布和资源隔离。

### 应用场景

- 企业级容器化应用部署
- 需要高可用性的关键业务系统
- 跨可用区容灾备份需求
- 大规模集群资源调度优化
- 边缘计算场景下的资源隔离需求

### 方案优势

- 自动化部署：使用Terraform实现基础设施即代码
- 高可用性：跨可用区部署提高系统稳定性
- 资源隔离：通过分区实现不同业务负载的物理隔离
- 网络优化：支持ENI网络模式，提高网络性能

### 涉及服务

- 弹性公网IP（EIP）：提供公网访问能力
- 虚拟私有云（VPC）：提供隔离的网络环境
- 云容器引擎（CCE）：提供容器集群管理服务

## 资源/数据源设计

本最佳实践涉及以下主要资源和数据源：

### 数据源

1. **云服务器规格（data.huaweicloud_compute_flavors）**
   - 用途：获取适用于节点的云服务器规格，用于创建CCE节点和节点池

### 资源

1. **VPC网络（huaweicloud_vpc）**
   - 用途：为CCE集群提供网络环境，实现网络隔离

2. **VPC子网（huaweicloud_vpc_subnet）**
   - 用途：在VPC中划分子网空间，包括普通子网和ENI网络专用子网

3. **CCE集群（huaweicloud_cce_cluster）**
   - 用途：创建和管理容器集群，配置ENI网络模式

4. **CCE分区（huaweicloud_cce_partition）**
   - 用途：创建CCE集群分区，实现资源隔离

5. **CCE节点（huaweicloud_cce_node）**
   - 用途：创建集群工作节点，与分区关联

6. **CCE节点池（huaweicloud_cce_node_pool）**
   - 用途：批量管理节点，与分区关联

### 资源/数据源依赖关系

```
data.huaweicloud_compute_flavors
    ├── huaweicloud_cce_node
    └── huaweicloud_cce_node_pool

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet (普通子网)
    │   └── huaweicloud_cce_cluster
    │       └── huaweicloud_cce_partition
    │           ├── huaweicloud_cce_node
    │           └── huaweicloud_cce_node_pool
    └── huaweicloud_vpc_subnet (ENI网络子网)
        └── huaweicloud_cce_partition
```

## 详细配置

### 数据源配置

#### 1. 云服务器规格（data.huaweicloud_compute_flavors）

获取默认region（默认继承当前provider块中所指定的region）下所有可用的云服务器规格信息，用于创建CCE节点和节点池。

```hcl
variable "iec_availability_zone" {
  description = "可用区名称，用于指定资源创建的可用区"
  type        = string
}

data "huaweicloud_compute_flavors" "test" {
  availability_zone = var.iec_availability_zone
  performance_type  = "computingv3"
  cpu_core_count    = 2
  memory_size       = 4
}
```

**参数说明**：
- **availability_zone**：可用区
- **performance_type**：性能类型
- **cpu_core_count**：CPU核心数
- **memory_size**：内存大小（GB）

### 资源配置

#### 1. VPC网络（huaweicloud_vpc）

在默认region（默认继承当前provider块中所指定的region）下创建VPC网络环境，为CCE集群提供网络隔离。

```hcl
variable "random_resource_name" {
  description = "资源名称，用于命名创建的各种资源"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  name = var.random_resource_name
  cidr = "192.168.0.0/16"
}
```

**参数说明**：
- **name**：VPC名称，使用变量动态生成
- **cidr**：VPC网段，格式为CIDR，如192.168.0.0/16

#### 2. VPC子网（huaweicloud_vpc_subnet）

在默认region（默认继承当前provider块中所指定的region）下创建两个子网，一个用于CCE集群基本网络，另一个用于ENI网络。

```hcl
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id = huaweicloud_vpc.test.id

  name       = var.random_resource_name
  cidr       = cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)

  primary_dns   = "100.125.1.250"
  secondary_dns = "100.125.21.250"
}

resource "huaweicloud_vpc_subnet" "eni_network" {
  vpc_id = huaweicloud_vpc.test.id

  name       = format("%s_eni_usage", var.random_resource_name)
  cidr       = cidrsubnet(huaweicloud_vpc.test.cidr, 8, 2)
  gateway_ip = cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 2), 1)

  availability_zone = var.iec_availability_zone
}
```

**参数说明**：
- **vpc_id**：VPC ID
- **name**：子网名称
- **cidr**：子网网段，使用cidrsubnet函数动态计算
- **gateway_ip**：网关IP，使用cidrhost函数动态计算
- **primary_dns**：主DNS服务器地址
- **secondary_dns**：备用DNS服务器地址
- **availability_zone**：可用区，用于ENI网络子网

#### 3. CCE集群（huaweicloud_cce_cluster）

在默认region（默认继承当前provider块中所指定的region）下创建CCE集群，配置ENI网络模式，为后续创建分区和节点提供基础环境。

```hcl
resource "huaweicloud_cce_cluster" "test" {
  name                   = var.random_resource_name
  flavor_id              = "cce.s1.small"
  vpc_id                 = huaweicloud_vpc.test.id
  subnet_id              = huaweicloud_vpc_subnet.test.id
  container_network_type = "eni"

  enable_distribute_management = true

  eni_subnet_id = join(",", [
    huaweicloud_vpc_subnet.test.ipv4_subnet_id,
  ])

  lifecycle {
    ignore_changes = [
      eni_subnet_id,
    ]
  }
}
```

**参数说明**：
- **name**：集群名称
- **flavor_id**：集群规格，这里使用cce.s1.small
- **vpc_id**：VPC ID
- **subnet_id**：子网ID
- **container_network_type**：容器网络类型，使用ENI模式
- **enable_distribute_management**：启用分布式管理
- **eni_subnet_id**：ENI子网ID
- **lifecycle**：生命周期管理，忽略eni_subnet_id的变化

#### 4. CCE分区（huaweicloud_cce_partition）

在默认region（默认继承当前provider块中所指定的region）下创建CCE分区，实现资源隔离，为节点和节点池提供分区环境。

```hcl
variable "iec_partition_border_group" {
  description = "分区边界组，用于指定分区的边界组"
  type        = string
}

resource "huaweicloud_cce_partition" "test" {
  cluster_id = huaweicloud_cce_cluster.test.id

  name                 = var.iec_availability_zone
  category             = "HomeZone"
  public_border_group  = var.iec_partition_border_group
  partition_subnet_id  = huaweicloud_vpc_subnet.eni_network.id
  container_subnet_ids = [huaweicloud_vpc_subnet.eni_network.ipv4_subnet_id]
}
```

**参数说明**：
- **cluster_id**：集群ID
- **name**：分区名称，使用可用区名称
- **category**：分区类别，设置为HomeZone
- **public_border_group**：公共边界组
- **partition_subnet_id**：分区子网ID
- **container_subnet_ids**：容器子网ID列表

#### 5. CCE节点（huaweicloud_cce_node）

在默认region（默认继承当前provider块中所指定的region）下创建CCE节点，与分区关联，提供容器运行环境。

```hcl
variable "node_password" {
  description = "节点登录密码，必须包含大小写字母、数字和特殊字符，长度8-26位"
  type        = string
  sensitive   = true
}

resource "huaweicloud_cce_node" "test" {
  cluster_id        = huaweicloud_cce_cluster.test.id
  name              = var.random_resource_name
  flavor_id         = try(data.huaweicloud_compute_flavors.test.flavors[0].id, "")
  availability_zone = var.iec_availability_zone
  partition         = huaweicloud_cce_partition.test.id
  password          = var.node_password

  root_volume {
    size       = 40
    volumetype = "SSD"
  }

  data_volumes {
    size       = 100
    volumetype = "SSD"
  }
}
```

**参数说明**：
- **cluster_id**：集群ID
- **name**：节点名称
- **flavor_id**：节点规格，从数据源获取
- **availability_zone**：可用区
- **partition**：分区ID，关联到创建的分区
- **password**：节点登录密码
- **root_volume**：系统盘配置
  - **size**：系统盘大小，单位为GB
  - **volumetype**：系统盘类型，如SSD
- **data_volumes**：数据盘配置
  - **size**：数据盘大小，单位为GB
  - **volumetype**：数据盘类型，如SSD

#### 6. CCE节点池（huaweicloud_cce_node_pool）

在默认region（默认继承当前provider块中所指定的region）下创建CCE节点池，与分区关联，实现节点的批量管理和弹性伸缩。

```hcl
resource "huaweicloud_cce_node_pool" "test" {
  cluster_id               = huaweicloud_cce_cluster.test.id
  name                     = var.random_resource_name
  flavor_id                = try(data.huaweicloud_compute_flavors.test.flavors[0].id, "")
  initial_node_count       = 1
  availability_zone        = var.iec_availability_zone
  password                 = var.node_password
  scall_enable             = false
  min_node_count           = 0
  max_node_count           = 0
  scale_down_cooldown_time = 0
  priority                 = 0
  type                     = "vm"
  partition                = huaweicloud_cce_partition.test.id

  root_volume {
    size       = 40
    volumetype = "SSD"
  }
  data_volumes {
    size       = 100
    volumetype = "SSD"
  }
}
```

**参数说明**：
- **cluster_id**：集群ID
- **name**：节点池名称
- **flavor_id**：节点规格，从数据源获取
- **initial_node_count**：初始节点数量
- **availability_zone**：可用区
- **password**：节点登录密码
- **scall_enable**：是否启用弹性伸缩
- **min_node_count**：最小节点数
- **max_node_count**：最大节点数
- **scale_down_cooldown_time**：缩容冷却时间
- **priority**：优先级
- **type**：节点类型
- **partition**：分区ID，关联到创建的分区
- **root_volume**：系统盘配置
  - **size**：系统盘大小，单位为GB
  - **volumetype**：系统盘类型，如SSD
- **data_volumes**：数据盘配置
  - **size**：数据盘大小，单位为GB
  - **volumetype**：数据盘类型，如SSD

## 部署流程

1. 创建VPC和子网（包括ENI网络专用子网）
2. 创建CCE集群，配置ENI网络模式
3. 创建CCE分区，关联ENI网络子网
4. 获取适用的云服务器规格
5. 创建CCE节点或节点池，关联到分区

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
   - 登录华为云控制台
   - 检查CCE集群状态
   - 验证分区创建情况
   - 确认节点已关联到正确的分区

## 注意事项

1. **成本控制**：
   - 选择合适的集群和节点规格
   - 合理规划节点数量

2. **网络规划**：
   - 合理规划IP地址段
   - 确保ENI网络子网与分区正确关联
   - 配置必要的安全组规则

3. **安全配置**：
   - 使用复杂密码保护节点安全
   - 启用安全组防护
   - 定期更新系统补丁

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 自动化部署的带有分区的CCE集群
2. 基于分区的资源隔离能力
3. 使用ENI网络模式的高性能容器网络
4. 可重复使用的Terraform配置

## 参考信息

- [华为云CCE产品文档](https://support.huaweicloud.com/cce/index.html)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE分区功能最佳实践](https://github.com/huaweicloud/terraform-provider-huaweicloud/blob/master/examples/cce/cce-with-partition)

