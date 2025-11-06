# 部署Turbo集群

## 应用场景

云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。Turbo集群是CCE提供的一种高性能集群类型，采用ENI（Elastic Network Interface）网络模式，提供更高的网络性能和更低的延迟，适用于对网络性能要求较高的生产环境。通过创建Turbo集群，可以快速部署和管理高性能容器化应用，实现微服务架构和DevOps实践。本最佳实践将介绍如何使用Terraform自动化部署一个CCE Turbo集群，包括可用区查询，以及VPC、子网、ENI子网、弹性公网IP和CCE集群的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [弹性公网IP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [CCE集群资源（huaweicloud_cce_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_cluster)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_vpc_subnet

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    │   └── huaweicloud_cce_cluster
    └── huaweicloud_vpc_subnet (ENI子网)
        └── huaweicloud_cce_cluster

huaweicloud_vpc_eip
    └── huaweicloud_cce_cluster
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询Turbo集群资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建Turbo集群相关资源：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建Turbo集群相关资源
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于为Turbo集群提供网络环境
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于为Turbo集群提供网络环境
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ENI子网资源，用于为Turbo集群提供ENI网络环境
resource "huaweicloud_vpc_subnet" "eni" {
  count = var.eni_ipv4_subnet_id == "" ? 1 : 0

  vpc_id            = var.vpc_id != "" ? var.vpc_id : huaweicloud_vpc.test[0].id
  name              = var.eni_subnet_name
  cidr              = var.eni_subnet_cidr != "" ? var.eni_subnet_cidr : cidrsubnet(huaweicloud_vpc.test[0].cidr, 4, 1)
  gateway_ip        = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : var.eni_subnet_cidr != "" ? cidrhost(var.eni_subnet_cidr, 1) : cidrhost(cidrsubnet(huaweicloud_vpc.test[0].cidr, 4, 1), 1)
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建ENI子网资源，仅当 `var.eni_ipv4_subnet_id` 为空时创建ENI子网资源
- **vpc_id**：ENI子网所属的VPC ID，如果指定了VPC ID则使用该值，否则引用VPC资源（huaweicloud_vpc.test[0]）的ID进行赋值
- **name**：ENI子网的名称，通过引用输入变量 `eni_subnet_name` 进行赋值
- **cidr**：ENI子网的CIDR块，如果指定了ENI子网CIDR则使用该值，否则基于VPC的CIDR块通过 `cidrsubnet` 函数自动计算（使用不同的子网索引以避免与普通子网冲突）
- **gateway_ip**：ENI子网的网关IP，如果指定了网关IP则使用该值，否则根据ENI子网CIDR或自动计算的ENI子网CIDR通过 `cidrhost` 函数自动计算
- **availability_zone**：ENI子网所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区

> 注意：ENI子网是Turbo集群特有的网络配置，用于提供高性能的网络连接。ENI子网必须与普通子网位于同一个VPC中，但需要使用不同的CIDR块以避免IP地址冲突。

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源，用于为Turbo集群提供公网访问能力
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE集群资源，用于部署和管理高性能容器化应用
resource "huaweicloud_cce_cluster" "test" {
  name                   = var.cluster_name
  flavor_id              = var.cluster_flavor_id
  cluster_version        = var.cluster_version
  cluster_type           = var.cluster_type
  container_network_type = var.container_network_type
  vpc_id                 = var.vpc_id != "" ? var.vpc_id : huaweicloud_vpc.test[0].id
  subnet_id              = var.subnet_id != "" ? var.subnet_id : huaweicloud_vpc_subnet.test[0].id
  eni_subnet_id          = var.eni_ipv4_subnet_id != "" ? var.eni_ipv4_subnet_id : huaweicloud_vpc_subnet.eni[0].ipv4_subnet_id
  eip                    = var.eip_address != "" ? var.eip_address : huaweicloud_vpc_eip.test[0].address
  description            = var.cluster_description
  tags                   = var.cluster_tags
}
```

**参数说明**：
- **name**：CCE集群的名称，通过引用输入变量 `cluster_name` 进行赋值
- **flavor_id**：CCE集群的规格ID，通过引用输入变量 `cluster_flavor_id` 进行赋值，默认为"cce.s1.small"表示小规格集群
- **cluster_version**：CCE集群的版本，通过引用输入变量 `cluster_version` 进行赋值，如果为null则使用最新版本
- **cluster_type**：CCE集群的类型，通过引用输入变量 `cluster_type` 进行赋值，默认为"VirtualMachine"表示虚拟机类型
- **container_network_type**：容器网络类型，通过引用输入变量 `container_network_type` 进行赋值，默认为"eni"表示ENI网络模式（Turbo集群必须使用ENI网络）
- **vpc_id**：VPC ID，如果指定了VPC ID则使用该值，否则引用VPC资源（huaweicloud_vpc.test[0]）的ID进行赋值
- **subnet_id**：子网ID，如果指定了子网ID则使用该值，否则引用VPC子网资源（huaweicloud_vpc_subnet.test[0]）的ID进行赋值
- **eni_subnet_id**：ENI子网ID，如果指定了ENI子网ID则使用该值，否则引用ENI子网资源（huaweicloud_vpc_subnet.eni[0]）的IPv4子网ID进行赋值
- **eip**：弹性公网IP地址，如果指定了EIP地址则使用该值，否则引用弹性公网IP资源（huaweicloud_vpc_eip.test[0]）的地址进行赋值
- **description**：集群的描述信息，通过引用输入变量 `cluster_description` 进行赋值
- **tags**：集群的标签，通过引用输入变量 `cluster_tags` 进行赋值，用于资源分类和管理

> 注意：Turbo集群必须使用ENI网络模式（`container_network_type`设置为"eni"），这是Turbo集群与Standard集群的主要区别。ENI网络模式提供更高的网络性能和更低的延迟，适用于对网络性能要求较高的生产环境。`eni_subnet_id`参数是Turbo集群特有的配置，必须指定一个独立的ENI子网。

### 8. 预设资源部署所需的入参

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
eni_subnet_name     = "tf_test_eni_subnet"
bandwidth_name      = "tf_test_bandwidth"
bandwidth_size      = 5
cluster_name        = "tf-test-cluster"
cluster_description = "Created by terraform script"
cluster_tags        = {
  owner = "terraform"
}
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

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Turbo集群
4. 运行 `terraform show` 查看已创建的Turbo集群

## 参考信息

- [华为云CCE产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE Turbo集群最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cce/turbo-cluster)
