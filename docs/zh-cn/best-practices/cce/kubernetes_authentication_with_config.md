# 部署Kubernetes并使用Config进行认证

## 应用场景

云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。在使用Kubernetes provider管理CCE集群中的Kubernetes资源时，需要正确配置Kubernetes provider以连接到CCE集群。本最佳实践将介绍如何使用Terraform自动化配置Kubernetes provider，通过将CCE集群的KubeConfig配置保存到本地文件，然后使用该配置文件来配置Kubernetes provider，实现与CCE集群的连接。这种方式适用于需要持久化KubeConfig配置或使用标准Kubernetes配置文件的场景。本最佳实践包括可用区、实例规格的查询，以及VPC、子网、弹性公网IP、CCE集群、节点等基础设施的创建，以及KubeConfig配置文件的生成和Kubernetes provider的配置。

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
- [本地文件资源（local_file）](https://registry.terraform.io/providers/hashicorp/local/latest/docs/resources/file)

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
                └── local_file

huaweicloud_vpc_eip
    └── huaweicloud_cce_cluster
        └── huaweicloud_cce_node
            └── local_file

huaweicloud_kps_keypair
    └── huaweicloud_cce_node
        └── local_file
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 配置Kubernetes Provider

由于本最佳实践需要使用Kubernetes provider来管理Kubernetes资源，需要在providers.tf文件中配置Kubernetes provider。在providers.tf文件中添加以下脚本：

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

# 配置Kubernetes provider，使用本地KubeConfig配置文件进行认证
provider "kubernetes" {
  config_path    = local_file.test.filename
  config_context = "external"
}
```

**参数说明**：
- **config_path**：Kubernetes配置文件的路径，引用本地文件资源（local_file.test）的文件名，该文件包含从CCE集群获取的KubeConfig配置
- **config_context**：Kubernetes配置上下文，设置为"external"表示使用外部访问上下文

> 注意：Kubernetes provider通过读取本地KubeConfig配置文件来连接到CCE集群。配置文件由local_file资源从CCE集群的kube_config_raw属性中获取并保存到本地。

### 3. 通过数据源查询资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建相关资源：

```hcl
variable "availability_zone" {
  description = "创建资源的可用区"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建相关资源
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询可用区信息，仅当 `var.availability_zone` 为空时查询可用区信息

### 4. 创建VPC资源（可选）

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于为集群提供网络环境
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于为集群提供网络环境
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源，用于为集群提供公网访问能力
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

### 8. 通过数据源查询节点资源创建所需的实例规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的实例规格：

```hcl
variable "node_flavor_id" {
  description = "节点的规格ID"
  type        = string
  default     = ""
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

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的实例规格信息，用于创建相关资源
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

  default = []
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
- **cluster_id**：CCE集群ID，引用CCE集群资源（huaweicloud_cce_cluster.test）的ID进行赋值
- **name**：节点的名称，通过引用输入变量 `node_name` 进行赋值
- **flavor_id**：节点规格ID，如果指定了节点规格ID则使用该值，否则使用实例规格列表查询数据源的第一个规格ID进行赋值
- **availability_zone**：节点所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区进行赋值
- **key_pair**：密钥对名称，引用密钥对资源（huaweicloud_kps_keypair.test）的名称进行赋值
- **root_volume**：根卷配置块
  - **volumetype**：根卷类型，通过引用输入变量 `root_volume_type` 进行赋值，默认为"SATA"
  - **size**：根卷大小（GB），通过引用输入变量 `root_volume_size` 进行赋值，默认为40GB
- **data_volumes**：数据卷配置块，通过动态块（dynamic block）根据输入变量 `data_volumes_configuration` 创建多个数据卷配置
  - **volumetype**：数据卷类型，通过输入变量 `data_volumes_configuration` 中的 `volumetype` 进行赋值
  - **size**：数据卷大小（GB），通过输入变量 `data_volumes_configuration` 中的 `size` 进行赋值

### 11. 创建本地KubeConfig配置文件

在TF文件中添加以下脚本以告知Terraform创建本地KubeConfig配置文件：

```hcl
# 创建本地KubeConfig配置文件，用于配置Kubernetes provider
resource "local_file" "test" {
  content  = huaweicloud_cce_cluster.test.kube_config_raw
  filename = ".kube/config"

  provisioner "local-exec" {
    command = "rm -rf .kube"
    when    = destroy
  }
}
```

**参数说明**：
- **content**：文件内容，引用CCE集群资源（huaweicloud_cce_cluster.test）的kube_config_raw属性，该属性包含完整的KubeConfig配置内容
- **filename**：文件路径，设置为".kube/config"，这是Kubernetes标准的配置文件路径
- **provisioner**：本地执行器配置块，用于在资源销毁时清理.kube目录
  - **command**：执行的命令，删除.kube目录
  - **when**：执行时机，设置为"destroy"表示在资源销毁时执行

> 注意：local_file资源将CCE集群的KubeConfig配置保存到本地文件，Kubernetes provider可以通过读取该文件来连接到CCE集群。provisioner用于在资源销毁时清理配置文件，避免留下敏感信息。

### 12. 预设资源部署所需的入参

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name                   = "tf_test_vpc"
subnet_name                = "tf_test_subnet"
bandwidth_name             = "tf_test_bandwidth"
bandwidth_size             = 5
cluster_name               = "tf-test-cluster"
node_performance_type      = "computingv3"
keypair_name               = "tf_test_keypair"
node_name                  = "tf-test-node"
root_volume_size           = 40
root_volume_type           = "SSD"
data_volumes_configuration = [
  {
    volumetype = "SSD"
    size       = 100
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

### 13. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境（这将下载Kubernetes provider和local provider）
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建资源和配置文件
4. 运行 `terraform show` 查看已创建的资源

> 注意：创建完成后，KubeConfig配置文件将保存在`.kube/config`文件中，Kubernetes provider将使用该文件连接到CCE集群。您可以使用该配置文件通过kubectl命令或其他Kubernetes工具管理集群资源。

## 参考信息

- [华为云CCE产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Kubernetes Provider文档](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs)
- [Local Provider文档](https://registry.terraform.io/providers/hashicorp/local/latest/docs)
- [CCE部署Kubernetes并使用Config进行认证最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cce/kubenetes/authenticate-with-config)
