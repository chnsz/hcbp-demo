# 部署按需计费集群

## 应用场景

华为云云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。通过部署按需计费的CCE集群，企业可以快速构建容器化应用环境，实现微服务架构部署，并根据实际使用量进行计费，有效控制成本。本最佳实践将介绍如何使用Terraform自动化部署按需计费的CCE集群，包括VPC、子网、CCE集群和节点的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [镜像查询数据源（data.huaweicloud_images_image）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_image)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [弹性公网IP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [密钥对资源（huaweicloud_compute_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_keypair)
- [CCE集群资源（huaweicloud_cce_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_cluster)
- [CCE节点资源（huaweicloud_cce_node）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_node)
- [计算实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [CCE节点挂载资源（huaweicloud_cce_node_attach）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_node_attach)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_cce_node
    └── huaweicloud_compute_instance

data.huaweicloud_images_image
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_cce_cluster
        │   ├── huaweicloud_cce_node
        │   └── huaweicloud_cce_node_attach
        └── huaweicloud_compute_instance

huaweicloud_vpc_eip
    └── huaweicloud_cce_cluster

huaweicloud_compute_keypair
    ├── huaweicloud_cce_node
    ├── huaweicloud_compute_instance
    └── huaweicloud_cce_node_attach

huaweicloud_compute_instance
    └── huaweicloud_cce_node_attach
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询CCE集群资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建CCE集群：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建CCE集群
data "huaweicloud_availability_zones" "myaz" {}
```

**参数说明**：
此数据源无需额外参数，默认查询当前区域下所有可用的可用区信息。

### 3. 通过数据源查询CCE集群资源创建所需的镜像

在TF文件中添加以下脚本以告知Terraform查询符合条件的镜像：

```hcl
variable "image_name" {
  description = "ECS镜像名称"
  type        = string
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的镜像信息，用于创建CCE集群
data "huaweicloud_images_image" "myimage" {
  name        = var.image_name
  most_recent = true
}
```

**参数说明**：
- **name**：镜像名称，通过引用输入变量image_name进行赋值
- **most_recent**：是否使用最新版本的镜像，设置为true表示使用最新版本

### 4. 创建VPC资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署CCE集群
resource "huaweicloud_vpc" "myvpc" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量vpc_cidr进行赋值

### 5. 创建VPC子网资源

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署CCE集群
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
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网网段，通过引用输入变量subnet_cidr进行赋值
- **gateway_ip**：网关IP地址，通过引用输入变量subnet_gateway进行赋值
- **primary_dns**：主DNS服务器地址，通过引用输入变量primary_dns进行赋值
- **secondary_dns**：备DNS服务器地址，通过引用输入变量secondary_dns进行赋值
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID

### 6. 创建弹性公网IP资源

在TF文件中添加以下脚本以告知Terraform创建弹性公网IP资源：

```hcl
variable "bandwidth_name" {
  description = "带宽名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源，用于部署CCE集群
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
- **type**：弹性公网IP类型，设置为"5_bgp"
- **name**：带宽名称，通过引用输入变量bandwidth_name进行赋值
- **size**：带宽大小（Mbit/s），设置为8Mbps
- **share_type**：带宽共享类型，设置为"PER"表示独享
- **charge_mode**：计费模式，设置为"traffic"表示按流量计费

### 7. 创建密钥对资源

在TF文件中添加以下脚本以告知Terraform创建密钥对资源：

```hcl
variable "key_pair_name" {
  description = "密钥对名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建密钥对资源，用于部署CCE集群
resource "huaweicloud_compute_keypair" "mykeypair" {
  name = var.key_pair_name
}
```

**参数说明**：
- **name**：密钥对名称，通过引用输入变量key_pair_name进行赋值

### 8. 创建CCE集群资源

在TF文件中添加以下脚本以告知Terraform创建CCE集群资源：

```hcl
variable "cce_cluster_name" {
  description = "CCE集群名称"
  type        = string
}

variable "cce_cluster_flavor" {
  description = "CCE集群规格"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE集群资源
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
- **name**：集群名称，通过引用输入变量cce_cluster_name进行赋值
- **flavor_id**：集群规格，通过引用输入变量cce_cluster_flavor进行赋值
- **vpc_id**：VPC的ID，引用前面创建的VPC资源的ID
- **subnet_id**：子网的ID，引用前面创建的子网资源的ID
- **container_network_type**：容器网络类型，设置为"overlay_l2"
- **eip**：弹性公网IP地址，引用前面创建的弹性公网IP资源的地址

### 9. 创建CCE节点资源

在TF文件中添加以下脚本以告知Terraform创建CCE节点资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE节点资源
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
- **cluster_id**：CCE集群的ID，引用前面创建的CCE集群资源的ID
- **name**：节点名称，通过引用输入变量node_name进行赋值
- **flavor_id**：节点规格，通过引用输入变量node_flavor进行赋值
- **availability_zone**：可用区，使用可用区列表查询数据源的第一个可用区
- **key_pair**：密钥对名称，引用前面创建的密钥对资源的名称
- **root_volume.size**：系统盘大小，通过引用输入变量root_volume_size进行赋值
- **root_volume.volumetype**：系统盘类型，通过引用输入变量root_volume_type进行赋值
- **data_volumes.size**：数据盘大小，通过引用输入变量data_volume_size进行赋值
- **data_volumes.volumetype**：数据盘类型，通过引用输入变量data_volume_type进行赋值

### 10. 创建计算实例资源

在TF文件中添加以下脚本以告知Terraform创建计算实例资源：

```hcl
variable "ecs_name" {
  description = "ECS实例名称"
  type        = string
}

variable "ecs_flavor" {
  description = "ECS实例规格"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建计算实例资源
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
- **name**：实例名称，通过引用输入变量ecs_name进行赋值
- **image_id**：镜像ID，使用镜像查询数据源的ID
- **flavor_id**：实例规格，通过引用输入变量ecs_flavor进行赋值
- **availability_zone**：可用区，使用可用区列表查询数据源的第一个可用区
- **key_pair**：密钥对名称，引用前面创建的密钥对资源的名称
- **delete_disks_on_termination**：实例删除时是否删除磁盘，设置为true
- **system_disk_type**：系统盘类型，通过引用输入变量root_volume_type进行赋值
- **system_disk_size**：系统盘大小，通过引用输入变量root_volume_size进行赋值
- **data_disks.type**：数据盘类型，通过引用输入变量data_volume_type进行赋值
- **data_disks.size**：数据盘大小，通过引用输入变量data_volume_size进行赋值
- **network.uuid**：网络ID，引用前面创建的子网资源的ID

### 11. 创建CCE节点挂载资源

在TF文件中添加以下脚本以告知Terraform创建CCE节点挂载资源：

```hcl
variable "os" {
  description = "操作系统类型"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE节点挂载资源
resource "huaweicloud_cce_node_attach" "test" {
  cluster_id = huaweicloud_cce_cluster.mycce.id
  server_id  = huaweicloud_compute_instance.myecs.id
  key_pair   = huaweicloud_compute_keypair.mykeypair.name
  os         = var.os
}
```

**参数说明**：
- **cluster_id**：CCE集群的ID，引用前面创建的CCE集群资源的ID
- **server_id**：ECS实例的ID，引用前面创建的计算实例资源的ID
- **key_pair**：密钥对名称，引用前面创建的密钥对资源的名称
- **os**：操作系统类型，通过引用输入变量os进行赋值

### 12. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 镜像配置
image_name = "Ubuntu 18.04 server 64bit"

# VPC配置
vpc_name = "tf_test_vpc"
vpc_cidr = "192.168.0.0/16"

# 子网配置
subnet_name = "tf_test_subnet"
subnet_cidr = "192.168.1.0/24"
subnet_gateway = "192.168.1.1"
primary_dns = "8.8.8.8"
secondary_dns = "8.8.4.4"

# 弹性公网IP配置
bandwidth_name = "tf_test_bandwidth"

# 密钥对配置
key_pair_name = "tf_test_keypair"

# CCE集群配置
cce_cluster_name = "tf_test_cluster"
cce_cluster_flavor = "cce.s1.small"

# CCE节点配置
node_name = "tf_test_node"
node_flavor = "s6.large.2"
root_volume_size = 40
root_volume_type = "SSD"
data_volume_size = 100
data_volume_type = "SSD"

# ECS实例配置
ecs_name = "tf_test_ecs"
ecs_flavor = "s6.large.2"

# 操作系统配置
os = "EulerOS 2.5"
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

### 13. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CCE集群
4. 运行 `terraform show` 查看已创建的CCE集群详情

## 参考信息

- [华为云CCE产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE集群最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cce)
