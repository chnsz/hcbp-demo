# 部署专属资源池

## 应用场景

魔坊（ModelArts）专属资源池为AI训练、推理和Notebook开发提供独享的计算资源，支持与VPC、SFS Turbo高性能文件存储及ModelArts网络的深度集成，满足企业级AI场景对网络隔离、存储性能和算力保障的需求。

本最佳实践适用于在ModelArts上创建专属资源池的场景，支持两种部署模式：一是同时创建VPC网络、SFS Turbo、ModelArts网络及专属资源池的完整部署；二是使用已有ModelArts网络及SFS Turbo连接信息，仅创建专属资源池。本最佳实践将介绍如何使用Terraform自动化部署上述资源，实现专属资源池的Infrastructure as Code管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ModelArts资源池规格列表查询数据源（data.huaweicloud_modelarts_resource_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelarts_resource_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [SFS Turbo文件系统资源（huaweicloud_sfs_turbo）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sfs_turbo)
- [ModelArts网络资源（huaweicloud_modelarts_network）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_network)
- [ModelArts工作空间资源（huaweicloud_modelarts_workspace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_workspace)
- [ModelArts专属资源池资源（huaweicloud_modelarts_resource_pool）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_resource_pool)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_sfs_turbo
    └── huaweicloud_modelarts_resource_pool

data.huaweicloud_modelarts_resource_flavors
    └── huaweicloud_modelarts_resource_pool

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
    └── huaweicloud_modelarts_network

huaweicloud_modelarts_network
    └── huaweicloud_modelarts_resource_pool

huaweicloud_modelarts_workspace
    └── huaweicloud_modelarts_resource_pool
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询可用区信息，其查询结果用于创建SFS Turbo文件系统及筛选专属资源池规格：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建SFS Turbo文件系统及专属资源池
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 此数据源无需额外参数，会自动查询当前区域的所有可用区

### 3. 创建VPC网络（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源。当需要新建SFS Turbo（即指定turbo_name）时创建：

```hcl
variable "turbo_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the SFS Turbo to create"

  validation {
    condition     = var.network_name != "" || var.turbo_name == ""
    error_message = "turbo_name can only be specified when network_name is specified."
  }
}

variable "vpc_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The VPC name. Can only be specified when turbo_name is specified."

  validation {
    condition     = (var.turbo_name != "") == (var.vpc_name != "")
    error_message = "vpc_name can only be specified when turbo_name is specified."
  }
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
  count = var.turbo_name != "" ? 1 : 0

  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **count**：资源的创建数，仅当turbo_name不为空时创建VPC
- **name**：VPC的名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null

> turbo_name仅能在同时指定network_name时使用；vpc_name必须与turbo_name同时指定或同时留空。

### 4. 创建VPC子网（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源。当需要新建SFS Turbo时创建：

```hcl
variable "subnet_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The subnet name. Can only be specified when turbo_name is specified."

  validation {
    condition     = (var.turbo_name != "") == (var.subnet_name != "")
    error_message = "subnet_name can only be specified when turbo_name is specified."
  }
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
  count = var.turbo_name != "" ? 1 : 0

  vpc_id     = huaweicloud_vpc.test[0].id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test[0].cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test[0].cidr, 8, 0), 1)
}
```

**参数说明**：
- **count**：资源的创建数，仅当turbo_name不为空时创建子网
- **vpc_id**：子网所属的VPC ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR块，当subnet_cidr不为空时使用subnet_cidr的值，否则基于VPC CIDR自动计算
- **gateway_ip**：子网的网关IP，当subnet_gateway_ip不为空时使用subnet_gateway_ip的值，否则基于子网CIDR自动计算

### 5. 创建安全组（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源。当需要新建SFS Turbo时创建：

```hcl
variable "security_group_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the security group"

  validation {
    condition     = (var.turbo_name != "") == (var.security_group_name != "")
    error_message = "security_group_name can only be specified when turbo_name is specified."
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  count = var.turbo_name != "" ? 1 : 0

  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **count**：资源的创建数，仅当turbo_name不为空时创建安全组
- **name**：安全组的名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认安全组规则

### 6. 创建安全组入方向TCP规则（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组入方向TCP规则，用于SFS Turbo访问。当需要新建SFS Turbo时创建：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组入方向TCP规则，用于SFS Turbo访问
# Make sure open the full ingress access for 111, 2048, 2049, 2051, 2052 and 20048 ports and about TCP and UDP protocols.
resource "huaweicloud_networking_secgroup_rule" "test" {
  count = var.turbo_name != "" ? 1 : 0

  security_group_id = huaweicloud_networking_secgroup.test[0].id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  ports             = "111,2048,2049,2051,2052,20048"
}
```

**参数说明**：
- **count**：资源的创建数，仅当turbo_name不为空时创建规则
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **direction**：规则方向，设置为"ingress"表示入方向
- **ethertype**：以太网类型，设置为"IPv4"
- **protocol**：协议类型，设置为"tcp"
- **ports**：开放的端口，设置为"111,2048,2049,2051,2052,20048"

> SFS Turbo需要开放111、2048、2049、2051、2052和20048端口的TCP入方向访问。实际部署时请遵循最小权限原则，根据业务需要调整端口和源地址范围。

### 7. 创建安全组入方向UDP规则（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组入方向UDP规则，用于SFS Turbo访问。当需要新建SFS Turbo时创建：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组入方向UDP规则，用于SFS Turbo访问
resource "huaweicloud_networking_secgroup_rule" "udp_ingress_access" {
  count = var.turbo_name != "" ? 1 : 0

  security_group_id = huaweicloud_networking_secgroup.test[0].id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "udp"
  ports             = "111,2048,2049,2051,2052,20048"
}
```

**参数说明**：
- **count**：资源的创建数，仅当turbo_name不为空时创建规则
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **direction**：规则方向，设置为"ingress"表示入方向
- **ethertype**：以太网类型，设置为"IPv4"
- **protocol**：协议类型，设置为"udp"
- **ports**：开放的端口，设置为"111,2048,2049,2051,2052,20048"

> SFS Turbo需要开放111、2048、2049、2051、2052和20048端口的UDP入方向访问。实际部署时请遵循最小权限原则，根据业务需要调整端口和源地址范围。

### 8. 创建SFS Turbo文件系统（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建SFS Turbo文件系统资源。当指定turbo_name时创建：

```hcl
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
  count = var.turbo_name != "" ? 1 : 0

  name                  = var.turbo_name
  size                  = var.turbo_size
  share_proto           = var.turbo_share_proto
  share_type            = var.turbo_share_type
  hpc_bandwidth         = var.turbo_hpc_bandwidth
  vpc_id                = huaweicloud_vpc.test[0].id
  subnet_id             = huaweicloud_vpc_subnet.test[0].id
  security_group_id     = huaweicloud_networking_secgroup.test[0].id
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
- **count**：资源的创建数，仅当turbo_name不为空时创建SFS Turbo
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

### 9. 创建ModelArts网络（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts网络资源。当指定network_name时创建；若使用已有网络，请配置network_id并留空network_name：

```hcl
variable "network_sfs_turbos" {
  type = list(object({
    id   = string
    name = string
  }))

  default     = []
  nullable    = false
  description = "The SFS Turbo connections for the ModelArts network"

  validation {
    condition     = length(var.network_sfs_turbos) == 0 || (var.network_name != "" && var.turbo_name == "")
    error_message = "network_sfs_turbos can only be specified when network_name is specified and turbo_name is not specified."
  }
}

variable "network_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the ModelArts network to create"

  validation {
    condition     = (var.network_id != "") != (var.network_name != "")
    error_message = "Exactly one of network_id and network_name must be specified."
  }
}

variable "network_cidr" {
  type        = string
  default     = "10.168.0.0/16"
  description = "The CIDR block of the ModelArts network"
}

variable "network_id" {
  type        = string
  default     = ""
  nullable    = false
  description = "The existing ModelArts network ID"
}

locals {
  network_sfs_turbos = var.turbo_name != "" ? [
    {
      id   = huaweicloud_sfs_turbo.test[0].id
      name = huaweicloud_sfs_turbo.test[0].name
    }
  ] : var.network_sfs_turbos
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts网络资源
resource "huaweicloud_modelarts_network" "test" {
  count = var.network_name != "" ? 1 : 0

  name = var.network_name
  cidr = var.network_cidr

  dynamic "sfs_turbos" {
    for_each = local.network_sfs_turbos

    content {
      id   = sfs_turbos.value.id
      name = sfs_turbos.value.name
    }
  }
}
```

**参数说明**：
- **count**：资源的创建数，仅当network_name不为空时创建ModelArts网络
- **name**：ModelArts网络的名称，通过引用输入变量network_name进行赋值
- **cidr**：ModelArts网络的CIDR块，通过引用输入变量network_cidr进行赋值，默认值为"10.168.0.0/16"
- **sfs_turbos.id**：关联的SFS Turbo ID，当新建SFS Turbo时引用创建的SFS Turbo资源ID，否则通过network_sfs_turbos变量指定
- **sfs_turbos.name**：关联的SFS Turbo名称，当新建SFS Turbo时引用创建的SFS Turbo资源名称，否则通过network_sfs_turbos变量指定

> network_id与network_name必须且只能指定其一。使用已有SFS Turbo时，可通过network_sfs_turbos指定连接信息，此时不能同时指定turbo_name。

### 10. 创建ModelArts工作空间（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts工作空间资源。当已通过workspace_id指定现有工作空间时可跳过此步骤：

```hcl
variable "workspace_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the workspace to create"

  validation {
    condition     = var.workspace_name == "" || var.workspace_id == ""
    error_message = "workspace_name and workspace_id cannot be configured at the same time."
  }
}

variable "workspace_id" {
  type        = string
  default     = ""
  description = "The existing workspace ID"
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

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询ModelArts专属资源池规格信息。当resource_pool_resources中所有资源的flavor_id均已指定时可跳过此步骤：

```hcl
variable "resource_pool_resources" {
  type = list(object({
    flavor_id     = optional(string, "")
    count         = number
    max_count     = optional(number)
    extend_params = optional(string)

    root_volume = optional(object({
      volume_type = string
      size        = string
    }))

    data_volumes = optional(list(object({
      volume_type   = string
      size          = string
      count         = optional(number)
      extend_params = optional(string)
    })), [])

    volume_group_configs = optional(list(object({
      volume_group     = string
      docker_thin_pool = optional(number)
      types            = optional(list(string))

      lvm_config = optional(object({
        lv_type = string
        path    = optional(string)
      }))
    })), [])

    os = optional(object({
      name       = optional(string)
      image_id   = optional(string)
      image_type = optional(string)
    }))

    driver = optional(object({
      version = string
    }))

    creating_step = optional(object({
      step = number
      type = string
    }))
  }))

  description = "The list of resource specifications in the resource pool"
}

locals {
  is_query_resource_flavor = length([for r in var.resource_pool_resources : r if r.flavor_id == ""]) > 0
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下ModelArts专属资源池规格信息，用于创建专属资源池
data "huaweicloud_modelarts_resource_flavors" "test" {
  count = local.is_query_resource_flavor ? 1 : 0

  type = "Dedicate"
}

locals {
  available_resource_flavors = [
    for o in try(data.huaweicloud_modelarts_resource_flavors.test[0].flavors, []) : o if lookup(o.az_status, try(data.huaweicloud_availability_zones.test.names[0], null), "soldout") == "normal"
  ]
}
```

**参数说明**：
- **count**：数据源的创建数，仅当resource_pool_resources中存在flavor_id为空的资源规格时执行规格查询
- **type**：资源池类型，设置为"Dedicate"表示专属资源池
- **is_query_resource_flavor**：本地变量，判断是否需要查询资源池规格
- **available_resource_flavors**：本地变量，筛选当前可用区状态为normal的专属资源池规格

### 12. 创建ModelArts专属资源池

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts专属资源池资源：

```hcl
variable "resource_pool_name" {
  type        = string
  description = "The name of the dedicated resource pool"
}

variable "resource_pool_description" {
  type        = string
  default     = null
  description = "The description of the dedicated resource pool"
}

variable "resource_pool_scope" {
  type        = list(string)
  default     = ["Train", "Infer", "Notebook"]
  description = "The scope of the dedicated resource pool"
}

variable "resource_pool_metadata_annotations" {
  type        = string
  default     = null
  description = "The annotations of the resource pool, in JSON format"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts专属资源池资源
resource "huaweicloud_modelarts_resource_pool" "test" {
  name         = var.resource_pool_name
  description  = var.resource_pool_description
  scope        = var.resource_pool_scope
  network_id   = var.network_id != "" ? var.network_id : huaweicloud_modelarts_network.test[0].id
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(huaweicloud_modelarts_workspace.test[0].id, null)

  dynamic "metadata" {
    for_each = var.resource_pool_metadata_annotations != null ? [1] : []

    content {
      annotations = var.resource_pool_metadata_annotations
    }
  }

  dynamic "resources" {
    for_each = var.resource_pool_resources

    content {
      flavor_id     = resources.value.flavor_id != "" ? resources.value.flavor_id : try(local.available_resource_flavors[0].id, null)
      count         = resources.value.count
      max_count     = resources.value.max_count
      extend_params = resources.value.extend_params

      dynamic "root_volume" {
        for_each = try(resources.value.root_volume, null) != null ? [resources.value.root_volume] : []

        content {
          volume_type = root_volume.value.volume_type
          size        = root_volume.value.size
        }
      }

      dynamic "data_volumes" {
        for_each = try(resources.value.data_volumes, [])

        content {
          volume_type   = data_volumes.value.volume_type
          size          = data_volumes.value.size
          extend_params = data_volumes.value.extend_params
          count         = try(data_volumes.value.count, null)
        }
      }

      dynamic "volume_group_configs" {
        for_each = try(resources.value.volume_group_configs, [])

        content {
          volume_group     = volume_group_configs.value.volume_group
          docker_thin_pool = try(volume_group_configs.value.docker_thin_pool, null)
          types            = try(volume_group_configs.value.types, null)

          dynamic "lvm_config" {
            for_each = volume_group_configs.value.lvm_config != null ? [volume_group_configs.value.lvm_config] : []

            content {
              lv_type = lvm_config.value.lv_type
              path    = try(lvm_config.value.path, null)
            }
          }
        }
      }

      dynamic "os" {
        for_each = resources.value.os != null ? [resources.value.os] : []

        content {
          name       = try(os.value.name, null)
          image_id   = try(os.value.image_id, null)
          image_type = try(os.value.image_type, null)
        }
      }

      dynamic "driver" {
        for_each = resources.value.driver != null ? [resources.value.driver] : []

        content {
          version = driver.value.version
        }
      }

      dynamic "creating_step" {
        for_each = resources.value.creating_step != null ? [resources.value.creating_step] : []

        content {
          step = creating_step.value.step
          type = creating_step.value.type
        }
      }
    }
  }
}
```

**参数说明**：
- **name**：专属资源池的名称，通过引用输入变量resource_pool_name进行赋值
- **description**：专属资源池的描述，通过引用输入变量resource_pool_description进行赋值，默认值为null
- **scope**：资源池的使用范围，通过引用输入变量resource_pool_scope进行赋值，默认值为["Train", "Infer", "Notebook"]
- **network_id**：ModelArts网络ID，当network_id不为空时使用其值，否则引用创建的ModelArts网络资源ID
- **workspace_id**：工作空间ID，当workspace_id不为空时使用其值，否则引用创建的工作空间资源ID
- **metadata.annotations**：资源池元数据注解，通过引用输入变量resource_pool_metadata_annotations进行赋值，JSON格式
- **resources.flavor_id**：资源规格ID，当flavor_id不为空时使用其值，否则从可用规格列表中选取第一个可用规格
- **resources.count**：资源数量，通过resource_pool_resources中对应项的count进行赋值
- **resources.max_count**：资源最大数量，通过resource_pool_resources中对应项的max_count进行赋值
- **resources.extend_params**：扩展参数，通过resource_pool_resources中对应项的extend_params进行赋值
- **resources.root_volume.volume_type**：系统盘类型，通过resource_pool_resources中对应项的root_volume.volume_type进行赋值
- **resources.root_volume.size**：系统盘大小，通过resource_pool_resources中对应项的root_volume.size进行赋值
- **resources.data_volumes.volume_type**：数据盘类型，通过resource_pool_resources中对应项的data_volumes.volume_type进行赋值
- **resources.data_volumes.size**：数据盘大小，通过resource_pool_resources中对应项的data_volumes.size进行赋值
- **resources.data_volumes.extend_params**：数据盘扩展参数，通过resource_pool_resources中对应项的data_volumes.extend_params进行赋值
- **resources.data_volumes.count**：数据盘数量，通过resource_pool_resources中对应项的data_volumes.count进行赋值
- **resources.volume_group_configs.volume_group**：卷组名称，通过resource_pool_resources中对应项的volume_group_configs.volume_group进行赋值
- **resources.volume_group_configs.docker_thin_pool**：Docker thin pool大小，通过resource_pool_resources中对应项的volume_group_configs.docker_thin_pool进行赋值
- **resources.volume_group_configs.types**：卷组类型列表，通过resource_pool_resources中对应项的volume_group_configs.types进行赋值
- **resources.volume_group_configs.lvm_config.lv_type**：LVM逻辑卷类型，通过resource_pool_resources中对应项的volume_group_configs.lvm_config.lv_type进行赋值
- **resources.volume_group_configs.lvm_config.path**：LVM逻辑卷路径，通过resource_pool_resources中对应项的volume_group_configs.lvm_config.path进行赋值
- **resources.os.name**：操作系统名称，通过resource_pool_resources中对应项的os.name进行赋值
- **resources.os.image_id**：操作系统镜像ID，通过resource_pool_resources中对应项的os.image_id进行赋值
- **resources.os.image_type**：操作系统镜像类型，通过resource_pool_resources中对应项的os.image_type进行赋值
- **resources.driver.version**：驱动版本，通过resource_pool_resources中对应项的driver.version进行赋值
- **resources.creating_step.step**：创建步骤序号，通过resource_pool_resources中对应项的creating_step.step进行赋值
- **resources.creating_step.type**：创建步骤类型，通过resource_pool_resources中对应项的creating_step.type进行赋值

### 13. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`terraform.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC网络配置
vpc_name            = "tf-test-vpc"
subnet_name         = "tf-test-subnet"
security_group_name = "tf-test-security-group"

# SFS Turbo配置
turbo_name = "tf-test-sfs-turbo"

# ModelArts网络与专属资源池配置
network_name              = "tf-test-network"
resource_pool_name        = "tf-test-resource-pool"
resource_pool_description = "This is a demo"

resource_pool_resources = [
  {
    count = 1
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="resource_pool_name=my-pool"`
2. 环境变量：`export TF_VAR_resource_pool_name=my-pool`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 14. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建专属资源池
4. 运行 `terraform show` 查看已创建的专属资源池详情

## 参考信息

- [华为云ModelArts产品文档](https://support.huaweicloud.com/modelarts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ModelArts专属资源池最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/modelarts/dedicated-resource-pool)
