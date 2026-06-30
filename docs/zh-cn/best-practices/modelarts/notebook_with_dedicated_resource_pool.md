# 部署专属资源池Notebook

## 应用场景

魔坊（ModelArts）Notebook是华为云提供的在线AI开发环境，支持Jupyter Notebook交互式开发，便于算法工程师进行数据探索、模型开发和调试。专属资源池为Notebook提供独享的计算资源，结合SFS Turbo高性能文件存储，满足企业级AI开发场景对网络隔离、存储性能和算力保障的需求。

本最佳实践适用于在ModelArts上创建专属资源池并部署Notebook实例的场景，涵盖VPC网络、SFS Turbo、ModelArts网络、工作空间、专属资源池、Notebook规格与镜像查询、密钥对及Notebook实例的完整部署流程。本最佳实践将介绍如何使用Terraform自动化部署上述资源，实现Notebook开发环境的Infrastructure as Code管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ModelArts资源池规格列表查询数据源（data.huaweicloud_modelarts_resource_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelarts_resource_flavors)
- [ModelArts Notebook规格列表查询数据源（data.huaweicloud_modelarts_notebook_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelarts_notebook_flavors)
- [ModelArts Notebook镜像列表查询数据源（data.huaweicloud_modelarts_notebook_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelarts_notebook_images)
- [ModelArts V2资源池列表查询数据源（data.huaweicloud_modelartsv2_resource_pools）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelartsv2_resource_pools)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [SFS Turbo文件系统资源（huaweicloud_sfs_turbo）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sfs_turbo)
- [ModelArts网络资源（huaweicloud_modelarts_network）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_network)
- [ModelArts工作空间资源（huaweicloud_modelarts_workspace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_workspace)
- [ModelArts专属资源池资源（huaweicloud_modelarts_resource_pool）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_resource_pool)
- [KPS密钥对资源（huaweicloud_kps_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [ModelArts Notebook资源（huaweicloud_modelarts_notebook）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_notebook)
- [ModelArts Notebook存储挂载资源（huaweicloud_modelarts_notebook_mount_storage）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_notebook_mount_storage)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_sfs_turbo
    └── huaweicloud_modelarts_resource_pool

data.huaweicloud_modelarts_resource_flavors
    └── huaweicloud_modelarts_resource_pool

data.huaweicloud_modelarts_notebook_flavors
    ├── data.huaweicloud_modelarts_notebook_images
    └── huaweicloud_modelarts_notebook

data.huaweicloud_modelarts_notebook_images
    └── huaweicloud_modelarts_notebook

data.huaweicloud_modelartsv2_resource_pools
    └── huaweicloud_modelarts_notebook

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_sfs_turbo

huaweicloud_vpc_subnet
    └── huaweicloud_sfs_turbo

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule.test
    ├── huaweicloud_networking_secgroup_rule.udp_ingress_access
    └── huaweicloud_sfs_turbo

huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_sfs_turbo

huaweicloud_networking_secgroup_rule.udp_ingress_access
    └── huaweicloud_sfs_turbo

huaweicloud_sfs_turbo
    ├── huaweicloud_modelarts_network
    └── huaweicloud_modelarts_notebook

huaweicloud_modelarts_network
    ├── huaweicloud_modelarts_resource_pool
    └── huaweicloud_modelarts_notebook

huaweicloud_modelarts_workspace
    └── huaweicloud_modelarts_resource_pool

huaweicloud_modelarts_resource_pool
    └── huaweicloud_modelarts_notebook

huaweicloud_kps_keypair
    └── huaweicloud_modelarts_notebook

huaweicloud_modelarts_notebook
    └── huaweicloud_modelarts_notebook_mount_storage
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询可用区信息，其查询结果用于创建SFS Turbo文件系统及专属资源池：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建SFS Turbo文件系统及专属资源池
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 此数据源无需额外参数，会自动查询当前区域的所有可用区

### 3. 创建VPC网络

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  type        = string
  description = "The VPC name"
}

variable "vpc_cidr" {
  type        = string
  default     = "192.168.0.0/16"
  description = "The CIDR block of the VPC"
}

variable "enterprise_project_id" {
  type        = string
  default     = null
  description = "The enterprise project ID"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源
resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **name**：VPC的名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null

### 4. 创建VPC子网

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  type        = string
  description = "The subnet name"
}

variable "subnet_cidr" {
  type        = string
  default     = ""
  nullable    = false
  description = "The CIDR block of the subnet"
}

variable "subnet_gateway_ip" {
  type        = string
  default     = ""
  nullable    = false
  description = "The gateway IP of the subnet"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR块，当subnet_cidr不为空时使用subnet_cidr的值，否则基于VPC CIDR自动计算
- **gateway_ip**：子网的网关IP，当subnet_gateway_ip不为空时使用subnet_gateway_ip的值，否则基于子网CIDR自动计算

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  type        = string
  description = "The name of the security group"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组的名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认安全组规则

### 6. 创建安全组入方向TCP规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组入方向TCP规则，用于SFS Turbo访问：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组入方向TCP规则，用于SFS Turbo访问
# Make sure open the full ingress access for 111, 2048, 2049, 2051, 2052 and 20048 ports and about TCP and UDP protocols.
resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  ports             = "111,2048,2049,2051,2052,20048"
}
```

**参数说明**：
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **direction**：规则方向，设置为"ingress"表示入方向
- **ethertype**：以太网类型，设置为"IPv4"
- **protocol**：协议类型，设置为"tcp"
- **ports**：开放的端口，设置为"111,2048,2049,2051,2052,20048"

> SFS Turbo需要开放111、2048、2049、2051、2052和20048端口的TCP入方向访问。实际部署时请遵循最小权限原则，根据业务需要调整端口和源地址范围。

### 7. 创建安全组入方向UDP规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组入方向UDP规则，用于SFS Turbo访问：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组入方向UDP规则，用于SFS Turbo访问
resource "huaweicloud_networking_secgroup_rule" "udp_ingress_access" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "udp"
  ports             = "111,2048,2049,2051,2052,20048"
}
```

**参数说明**：
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **direction**：规则方向，设置为"ingress"表示入方向
- **ethertype**：以太网类型，设置为"IPv4"
- **protocol**：协议类型，设置为"udp"
- **ports**：开放的端口，设置为"111,2048,2049,2051,2052,20048"

> SFS Turbo需要开放111、2048、2049、2051、2052和20048端口的UDP入方向访问。实际部署时请遵循最小权限原则，根据业务需要调整端口和源地址范围。

### 8. 创建SFS Turbo文件系统

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建SFS Turbo文件系统资源：

```hcl
variable "turbo_name" {
  type        = string
  description = "The name of the SFS Turbo"
}

variable "turbo_size" {
  type        = number
  default     = 1228
  description = "The size of the SFS Turbo"
}

variable "turbo_share_proto" {
  type        = string
  default     = "NFS"
  description = "The share protocol of the SFS Turbo"
}

variable "turbo_share_type" {
  type        = string
  default     = "HPC"
  description = "The share type of the SFS Turbo"
}

variable "turbo_hpc_bandwidth" {
  type        = string
  default     = "40M"
  description = "The HPC bandwidth of the SFS Turbo"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SFS Turbo文件系统资源
resource "huaweicloud_sfs_turbo" "test" {
  name                  = var.turbo_name
  size                  = var.turbo_size
  share_proto           = var.turbo_share_proto
  share_type            = var.turbo_share_type
  hpc_bandwidth         = var.turbo_hpc_bandwidth
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zone     = try(data.huaweicloud_availability_zones.test.names[0], null)
  enterprise_project_id = var.enterprise_project_id

  depends_on = [
    huaweicloud_networking_secgroup_rule.test,
    huaweicloud_networking_secgroup_rule.udp_ingress_access,
  ]

  lifecycle {
    ignore_changes = [
      availability_zone,
    ]
  }
}
```

**参数说明**：
- **name**：SFS Turbo的名称，通过引用输入变量turbo_name进行赋值
- **size**：SFS Turbo的容量，通过引用输入变量turbo_size进行赋值，默认值为1228
- **share_proto**：共享协议，通过引用输入变量turbo_share_proto进行赋值，默认值为"NFS"
- **share_type**：共享类型，通过引用输入变量turbo_share_type进行赋值，默认值为"HPC"
- **hpc_bandwidth**：HPC带宽，通过引用输入变量turbo_hpc_bandwidth进行赋值，默认值为"40M"
- **vpc_id**：VPC ID，引用前面创建的VPC资源的ID
- **subnet_id**：子网ID，引用前面创建的VPC子网资源的ID
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **availability_zone**：可用区，根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值
- **depends_on**：显式依赖安全组规则资源，确保安全组规则创建完成后再创建SFS Turbo
- **lifecycle.ignore_changes**：生命周期管理，忽略availability_zone的变更

### 9. 创建ModelArts网络

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts网络资源：

```hcl
variable "network_name" {
  type        = string
  description = "The name of the network"
}

variable "network_cidr" {
  type        = string
  default     = "10.168.0.0/16"
  description = "The CIDR block of the network"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts网络资源
resource "huaweicloud_modelarts_network" "test" {
  name = var.network_name
  cidr = var.network_cidr

  sfs_turbos {
    name = huaweicloud_sfs_turbo.test.name
    id   = huaweicloud_sfs_turbo.test.id
  }
}
```

**参数说明**：
- **name**：ModelArts网络的名称，通过引用输入变量network_name进行赋值
- **cidr**：ModelArts网络的CIDR块，通过引用输入变量network_cidr进行赋值，默认值为"10.168.0.0/16"
- **sfs_turbos.name**：关联的SFS Turbo名称，引用前面创建的SFS Turbo资源的名称
- **sfs_turbos.id**：关联的SFS Turbo ID，引用前面创建的SFS Turbo资源的ID

### 10. 创建ModelArts工作空间（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts工作空间资源。当已通过workspace_id指定现有工作空间时可跳过此步骤：

```hcl
variable "workspace_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the workspace to create. Cannot be configured together with workspace_id."

  validation {
    condition     = var.workspace_name == "" || var.workspace_id == ""
    error_message = "workspace_name and workspace_id cannot be configured at the same time."
  }
}

variable "workspace_id" {
  type        = string
  default     = ""
  description = "The existing workspace ID. Cannot be configured together with workspace_name."
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts工作空间资源
resource "huaweicloud_modelarts_workspace" "test" {
  count = var.workspace_name != "" ? 1 : 0

  name = var.workspace_name
}
```

**参数说明**：
- **count**：资源的创建数，仅当workspace_name不为空时创建工作空间
- **name**：工作空间的名称，通过引用输入变量workspace_name进行赋值

> workspace_name与workspace_id不能同时配置。若使用已有工作空间，请配置workspace_id并留空workspace_name。

### 11. 查询ModelArts专属资源池规格

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询ModelArts专属资源池规格信息。当已通过resource_pool_flavor_id指定规格ID时可跳过此步骤：

```hcl
variable "resource_pool_flavor_id" {
  type        = string
  default     = ""
  nullable    = false
  description = "The flavor ID of the resource pool"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下ModelArts专属资源池规格信息，用于创建专属资源池
data "huaweicloud_modelarts_resource_flavors" "test" {
  count = var.resource_pool_flavor_id != "" ? 0 : 1

  type = "Dedicate"
}
```

**参数说明**：
- **count**：数据源的创建数，仅当resource_pool_flavor_id为空时执行规格查询
- **type**：资源池类型，设置为"Dedicate"表示专属资源池

### 12. 创建ModelArts专属资源池

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts专属资源池资源：

```hcl
variable "resource_pool_name" {
  type        = string
  description = "The name of the resource pool"
}

variable "resource_pool_scope" {
  type        = list(string)
  default     = ["Notebook", "Train", "Infer"]
  description = "The scope of the resource pool"
}

locals {
  available_resource_flavors = [
    for o in try(data.huaweicloud_modelarts_resource_flavors.test[0].flavors, []) : o if lookup(o.az_status, try(data.huaweicloud_availability_zones.test.names[0], null), "soldout") == "normal"
  ]
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts专属资源池资源
resource "huaweicloud_modelarts_resource_pool" "test" {
  name         = var.resource_pool_name
  scope        = var.resource_pool_scope
  network_id   = huaweicloud_modelarts_network.test.id
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(huaweicloud_modelarts_workspace.test[0].id, null)

  resources {
    flavor_id = var.resource_pool_flavor_id != "" ? var.resource_pool_flavor_id : try(local.available_resource_flavors[0].id, null)
    count     = 1
  }

  # If you want to change the `flavor` or other fields, you need to remove the corresponding fields from "lifecycle.ignore_changes".
  lifecycle {
    ignore_changes = [
      resources,
    ]
  }
}
```

**参数说明**：
- **name**：专属资源池的名称，通过引用输入变量resource_pool_name进行赋值
- **scope**：资源池的使用范围，通过引用输入变量resource_pool_scope进行赋值，默认值为["Notebook", "Train", "Infer"]
- **network_id**：ModelArts网络ID，引用前面创建的ModelArts网络资源的ID
- **workspace_id**：工作空间ID，当workspace_id不为空时使用其值，否则引用创建的工作空间资源ID
- **resources.flavor_id**：资源池规格ID，当resource_pool_flavor_id不为空时使用其值，否则从可用规格列表中选取第一个可用规格
- **resources.count**：资源数量，设置为1
- **lifecycle.ignore_changes**：生命周期管理，忽略resources的变更。如需修改flavor或其他字段，需从ignore_changes中移除对应字段

### 13. 查询ModelArts Notebook规格

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询ModelArts Notebook规格信息。当已通过notebook_flavor_id指定规格ID时可跳过此步骤：

```hcl
variable "notebook_flavor_id" {
  type        = string
  default     = ""
  nullable    = false
  description = "The flavor ID of the notebook. If empty, the first available dedicated flavor is used"
}

variable "notebook_flavor_category" {
  type        = string
  default     = "CPU"
  description = "The processor type of the notebook flavor"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下ModelArts Notebook规格信息，用于创建Notebook实例
data "huaweicloud_modelarts_notebook_flavors" "test" {
  count = var.notebook_flavor_id != "" ? 0 : 1

  type     = "DEDICATED"
  category = var.notebook_flavor_category
}

locals {
  available_notebook_flavors = [for o in try(data.huaweicloud_modelarts_notebook_flavors.test[0].flavors, []) : o if !o.sold_out]
}
```

**参数说明**：
- **count**：数据源的创建数，仅当notebook_flavor_id为空时执行规格查询
- **type**：Notebook类型，设置为"DEDICATED"表示专属资源池Notebook
- **category**：处理器类型，通过引用输入变量notebook_flavor_category进行赋值，默认值为"CPU"
- **available_notebook_flavors**：本地变量，筛选未售罄的Notebook规格

### 14. 查询ModelArts Notebook镜像

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询ModelArts Notebook镜像信息。当已通过notebook_image_id指定镜像ID时可跳过此步骤：

```hcl
variable "notebook_image_id" {
  type        = string
  default     = ""
  nullable    = false
  description = "The image ID of the notebook."
}

variable "notebook_image_type" {
  type        = string
  default     = "BUILD_IN"
  description = "The type of the notebook image"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下ModelArts Notebook镜像信息，用于创建Notebook实例
data "huaweicloud_modelarts_notebook_images" "test" {
  count = var.notebook_image_id != "" ? 0 : 1

  type     = var.notebook_image_type
  cpu_arch = try(data.huaweicloud_modelarts_notebook_flavors.test[0].flavors[0].arch, "x86_64")
}

locals {
  available_notebook_images = [
    for o in try(data.huaweicloud_modelarts_notebook_images.test[0].images, []) : o if contains(o.resource_categories, var.notebook_flavor_category) && o.status == "ACTIVE" && contains(o.dev_services, "NOTEBOOK")
  ]
}
```

**参数说明**：
- **count**：数据源的创建数，仅当notebook_image_id为空时执行镜像查询
- **type**：镜像类型，通过引用输入变量notebook_image_type进行赋值，默认值为"BUILD_IN"
- **cpu_arch**：CPU架构，根据Notebook规格查询数据源返回的第一个规格的arch进行赋值，默认值为"x86_64"
- **available_notebook_images**：本地变量，筛选状态为ACTIVE且支持NOTEBOOK服务的镜像

### 15. 创建KPS密钥对（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建KPS密钥对资源，用于Notebook远程SSH访问。当已配置notebook_key_pair_name或未配置allowed_access_ips时可跳过此步骤：

```hcl
variable "notebook_key_pair_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The key pair name for remote SSH access"

  validation {
    condition     = length(var.allowed_access_ips) == 0 || var.notebook_key_pair_name != "" || var.keypair_name != ""
    error_message = "When allowed_access_ips is configured, either notebook_key_pair_name or keypair_name must be specified."
  }
}

variable "allowed_access_ips" {
  type        = list(string)
  default     = []
  nullable    = false
  description = "The public IP addresses that are allowed for remote SSH access"
}

variable "keypair_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the KPS key pair"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建KPS密钥对资源
resource "huaweicloud_kps_keypair" "test" {
  count = var.notebook_key_pair_name == "" && length(var.allowed_access_ips) > 0 ? 1 : 0

  name = var.keypair_name
}
```

**参数说明**：
- **count**：资源的创建数，仅当notebook_key_pair_name为空且allowed_access_ips不为空时创建密钥对
- **name**：密钥对名称，通过引用输入变量keypair_name进行赋值

> 配置allowed_access_ips时，必须指定notebook_key_pair_name或keypair_name。

### 16. 查询ModelArts V2资源池信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询ModelArts V2资源池信息，用于获取Notebook工作空间ID：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下ModelArts V2资源池信息，用于获取Notebook工作空间ID
data "huaweicloud_modelartsv2_resource_pools" "test" {}

locals {
  resource_pool = try([for pool in data.huaweicloud_modelartsv2_resource_pools.test.resource_pools : pool if pool.metadata[0].name ==
  huaweicloud_modelarts_resource_pool.test.id][0], {})
}
```

**参数说明**：
- 此数据源无需额外参数，会自动查询当前区域的所有V2资源池
- **resource_pool**：本地变量，根据专属资源池ID匹配对应的V2资源池信息

### 17. 创建ModelArts Notebook实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts Notebook实例资源：

```hcl
variable "notebook_name" {
  type        = string
  description = "The name of the notebook"
}

variable "notebook_description" {
  type        = string
  default     = null
  description = "The description of the notebook"
}

variable "notebook_tags" {
  type        = map(string)
  default     = {}
  description = "The tags of the notebook"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts Notebook实例资源
resource "huaweicloud_modelarts_notebook" "test" {
  name               = var.notebook_name
  flavor_id          = var.notebook_flavor_id != "" ? var.notebook_flavor_id : try(local.available_notebook_flavors[0].id, null)
  image_id           = var.notebook_image_id != "" ? var.notebook_image_id : try(local.available_notebook_images[0].id, null)
  description        = var.notebook_description
  pool_id            = huaweicloud_modelarts_resource_pool.test.id
  workspace_id       = try(local.resource_pool.metadata[0].labels["os.modelarts/workspace.id"], null)
  key_pair           = var.notebook_key_pair_name != "" ? var.notebook_key_pair_name : try(huaweicloud_kps_keypair.test[0].name, null)
  allowed_access_ips = length(var.allowed_access_ips) > 0 ? var.allowed_access_ips : null

  volume {
    type      = "EFS"
    ownership = "DEDICATED"
    uri       = format("%s:/", try(huaweicloud_modelarts_network.test.sfs_turbos[0].uri, null))
    id        = huaweicloud_sfs_turbo.test.id
  }

  tags = var.notebook_tags
}
```

**参数说明**：
- **name**：Notebook的名称，通过引用输入变量notebook_name进行赋值
- **flavor_id**：Notebook规格ID，当notebook_flavor_id不为空时使用其值，否则从可用规格列表中选取第一个
- **image_id**：Notebook镜像ID，当notebook_image_id不为空时使用其值，否则从可用镜像列表中选取第一个
- **description**：Notebook描述，通过引用输入变量notebook_description进行赋值
- **pool_id**：专属资源池ID，引用前面创建的专属资源池资源的ID
- **workspace_id**：工作空间ID，从V2资源池信息的metadata.labels中获取
- **key_pair**：密钥对名称，当notebook_key_pair_name不为空时使用其值，否则引用创建的KPS密钥对资源名称
- **allowed_access_ips**：允许远程SSH访问的公网IP列表，通过引用输入变量allowed_access_ips进行赋值
- **volume**：存储卷配置，类型为EFS，ownership为DEDICATED，关联SFS Turbo
- **tags**：Notebook标签，通过引用输入变量notebook_tags进行赋值，默认值为空映射

### 18. 创建ModelArts Notebook存储挂载（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts Notebook存储挂载资源。当notebook_mount_storage_path和notebook_mount_storage_local_directory均不为空时创建：

```hcl
variable "notebook_mount_storage_path" {
  type        = string
  default     = ""
  nullable    = false
  description = "The OBS path of Parallel File System (PFS) or its folders to mount"
}

variable "notebook_mount_storage_local_directory" {
  type        = string
  default     = ""
  nullable    = false
  description = "The local mount directory for the OBS storage"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts Notebook存储挂载资源
resource "huaweicloud_modelarts_notebook_mount_storage" "test" {
  count = var.notebook_mount_storage_path != "" && var.notebook_mount_storage_local_directory != "" ? 1 : 0

  notebook_id           = huaweicloud_modelarts_notebook.test.id
  storage_path          = var.notebook_mount_storage_path
  local_mount_directory = var.notebook_mount_storage_local_directory
}
```

**参数说明**：
- **count**：资源的创建数，仅当notebook_mount_storage_path和notebook_mount_storage_local_directory均不为空时创建
- **notebook_id**：Notebook ID，引用前面创建的Notebook资源的ID
- **storage_path**：并行文件系统（PFS）或其文件夹的OBS路径，通过引用输入变量notebook_mount_storage_path进行赋值
- **local_mount_directory**：本地挂载目录，通过引用输入变量notebook_mount_storage_local_directory进行赋值

### 19. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`terraform.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC网络配置
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"

# SFS Turbo配置
turbo_name = "tf_test_sfs_turbo"

# ModelArts网络与资源池配置
network_name       = "tf-test-network"
resource_pool_name = "tf-test-resource-pool"

# Notebook配置
notebook_name        = "tf_test_notebook"
notebook_description = "Created by Terraform script"

notebook_tags = {
  owner = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="notebook_name=my-notebook"`
2. 环境变量：`export TF_VAR_notebook_name=my-notebook`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 20. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建专属资源池及Notebook实例
4. 运行 `terraform show` 查看已创建的专属资源池及Notebook实例详情

## 参考信息

- [华为云ModelArts产品文档](https://support.huaweicloud.com/modelarts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ModelArts专属资源池Notebook最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/modelarts/notebook-with-dedicated-resource-pool)
