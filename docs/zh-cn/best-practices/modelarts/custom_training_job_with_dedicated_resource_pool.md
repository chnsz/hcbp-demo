# 部署专属资源池自定义训练作业

## 应用场景

魔坊（ModelArts）是华为云提供的面向AI开发者的模型训推平台，支持从数据处理、算法开发、模型训练到模型部署的全流程AI开发。专属资源池为ModelArts提供独享的计算资源，适用于对算力隔离、网络配置和存储性能有较高要求的企业级AI训练场景。

本最佳实践适用于需要在ModelArts上创建专属资源池并提交自定义训练作业的场景，涵盖VPC网络、SFS Turbo高性能文件存储、ModelArts网络、工作空间、专属资源池及训练作业的完整部署流程。本最佳实践将介绍如何使用Terraform自动化部署上述资源，实现端到端的AI训练基础设施编排。

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
- [SMN主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [ModelArts训练作业资源（huaweicloud_modelarts_training_job）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_training_job)

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
    ├── huaweicloud_modelarts_resource_pool
    └── huaweicloud_modelarts_training_job

huaweicloud_modelarts_resource_pool
    └── huaweicloud_modelarts_training_job

huaweicloud_smn_topic
    └── huaweicloud_modelarts_training_job
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
  cidr = var.network_cidr # The recommended connecting CIDR about SFS Turbo.

  sfs_turbos {
    name = huaweicloud_sfs_turbo.test.name
    id   = huaweicloud_sfs_turbo.test.id
  }
}
```

**参数说明**：
- **name**：ModelArts网络的名称，通过引用输入变量network_name进行赋值
- **cidr**：ModelArts网络的CIDR块，通过引用输入变量network_cidr进行赋值，默认值为"10.168.0.0/16"，建议使用与SFS Turbo对接的推荐CIDR
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

### 13. 创建SMN主题（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建SMN主题资源，用于训练作业通知。当已通过training_job_notification_topic_urn指定现有主题时可跳过此步骤：

```hcl
variable "topic_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the SMN topic to create for notifications. Cannot be configured together with training_job_notification_topic_urn."

  validation {
    condition     = var.topic_name == "" || var.training_job_notification_topic_urn == ""
    error_message = "topic_name and training_job_notification_topic_urn cannot be configured at the same time."
  }
}

variable "training_job_notification_topic_urn" {
  type        = string
  default     = ""
  nullable    = false
  description = "The existing SMN topic URN for training job notifications. Cannot be configured together with topic_name."
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN主题资源
resource "huaweicloud_smn_topic" "test" {
  count = var.topic_name != "" ? 1 : 0

  name = var.topic_name
}
```

**参数说明**：
- **count**：资源的创建数，仅当topic_name不为空时创建SMN主题
- **name**：SMN主题的名称，通过引用输入变量topic_name进行赋值

> topic_name与training_job_notification_topic_urn不能同时配置。

### 14. 创建ModelArts自定义训练作业

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts自定义训练作业资源：

```hcl
variable "training_job_name" {
  type        = string
  description = "The name of the training job"
}

variable "training_job_annotations" {
  type        = map(string)
  default     = {}
  description = "The annotations of the training job"
}

variable "training_job_description" {
  type        = string
  default     = null
  description = "The description of the training job"
}

variable "training_job_code_dir" {
  type        = string
  description = "The OBS code directory of the training job"
}

variable "training_job_command" {
  type        = string
  description = "The container startup command for the training job"
}

variable "training_job_engine" {
  type = object({
    image_url = optional(string)
    id        = optional(string)
    version   = optional(string)
    name      = optional(string)
  })

  description = "The engine configuration of the training job"
}

variable "training_job_inputs" {
  type = list(object({
    local_dir = string

    dataset = object({
      id           = string
      name         = optional(string)
      version_id   = optional(string)
      service_type = optional(string)
    })
  }))

  default     = []
  nullable    = false
  description = "The inputs of the training job"
}

variable "training_job_environments" {
  type        = map(string)
  default     = {}
  nullable    = false
  description = "The environment variables of the training job"
}

variable "resource_node_count" {
  type        = number
  default     = 1
  description = "The number of resource replicas used by the training job"
}

variable "training_job_volumes" {
  type = list(object({
    nfs = optional(object({
      nfs_server_path = optional(string)
      local_path      = optional(string)
      read_only       = optional(bool)
    }))

    pfs = optional(object({
      pfs_path   = optional(string)
      local_path = optional(string)
    }))

    obs = optional(object({
      obs_path   = optional(string)
      local_path = optional(string)
    }))
  }))

  default     = []
  nullable    = false
  description = "The volume mount configuration of the training job"
}

variable "training_job_log_export_path_obs_url" {
  type        = string
  default     = ""
  nullable    = false
  description = "The log export path of the training job"
}

variable "training_job_log_export_config_version" {
  type        = string
  default     = ""
  nullable    = false
  description = "The log export config version of the training job"
}

variable "training_job_auto_stop_duration" {
  type        = number
  default     = 0
  description = "The auto stop duration of the training job"
}

variable "training_job_notification_events" {
  type        = list(string)
  default     = []
  description = "The notification events of the training job"
}

variable "custom_metrics" {
  type = object({
    exec = optional(object({
      command = list(string)
    }))

    http_get = optional(object({
      path = string
      port = number
    }))
  })

  default     = null
  description = "The custom metrics of the training job"
}

variable "training_job_asset_model" {
  type = object({
    name    = string
    version = string
    type    = string
    code    = optional(string)
    desc    = optional(string)
    series  = optional(string)
  })

  default     = null
  description = "The asset model of the training job"
}

variable "training_job_output_model" {
  type = object({
    obs_path   = string
    local_path = optional(string)
  })

  default     = null
  description = "The output model of the training job"
}

variable "training_job_tags" {
  type        = map(string)
  default     = {}
  description = "The tags of the training job"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts自定义训练作业资源
resource "huaweicloud_modelarts_training_job" "test" {
  kind = "job"

  metadata {
    name         = var.training_job_name
    workspace_id = var.workspace_id != "" ? var.workspace_id : try(huaweicloud_modelarts_workspace.test[0].id, null)
    annotations  = var.training_job_annotations
    description  = var.training_job_description
  }

  algorithm {
    code_dir = var.training_job_code_dir
    command  = var.training_job_command

    engine {
      image_url      = var.training_job_engine.image_url
      engine_id      = var.training_job_engine.id
      engine_version = var.training_job_engine.version
      engine_name    = var.training_job_engine.name
    }


    dynamic "inputs" {
      for_each = var.training_job_inputs

      content {
        local_dir = inputs.value.local_dir

        remote {
          dataset {
            id           = inputs.value.dataset.id
            name         = inputs.value.dataset.name
            version_id   = inputs.value.dataset.version_id
            service_type = inputs.value.dataset.service_type
          }
        }
      }
    }

    environments = var.training_job_environments
  }

  spec {
    resource {
      pool_id    = huaweicloud_modelarts_resource_pool.test.id
      node_count = var.resource_node_count
    }

    dynamic "volumes" {
      for_each = var.training_job_volumes

      content {
        dynamic "nfs" {
          for_each = volumes.value.nfs != null ? [volumes.value.nfs] : []

          content {
            nfs_server_path = nfs.value.nfs_server_path
            local_path      = nfs.value.local_path
            read_only       = nfs.value.read_only
          }
        }

        dynamic "pfs" {
          for_each = volumes.value.pfs != null ? [volumes.value.pfs] : []

          content {
            pfs_path   = pfs.value.pfs_path
            local_path = pfs.value.local_path
          }
        }

        dynamic "obs" {
          for_each = volumes.value.obs != null ? [volumes.value.obs] : []

          content {
            obs_path   = obs.value.obs_path
            local_path = obs.value.local_path
          }
        }
      }
    }

    dynamic "log_export_path" {
      for_each = var.training_job_log_export_path_obs_url != "" ? [1] : []

      content {
        obs_url = var.training_job_log_export_path_obs_url
      }
    }

    dynamic "log_export_config" {
      for_each = var.training_job_log_export_config_version != "" ? [1] : []

      content {
        version = var.training_job_log_export_config_version
      }
    }

    dynamic "auto_stop" {
      for_each = var.training_job_auto_stop_duration != 0 ? [1] : []

      content {
        time_unit = "HOURS"
        duration  = var.training_job_auto_stop_duration
      }
    }

    dynamic "notification" {
      for_each = var.training_job_notification_topic_urn != "" || var.topic_name != "" ? [1] : []

      content {
        topic_urn = var.training_job_notification_topic_urn != "" ? var.training_job_notification_topic_urn : var.topic_name != "" ? huaweicloud_smn_topic.test[0].id : ""
        events    = var.training_job_notification_events
      }
    }

    dynamic "custom_metrics" {
      for_each = var.custom_metrics != null ? [var.custom_metrics] : []

      content {
        dynamic "exec" {
          for_each = custom_metrics.value.exec != null ? [custom_metrics.value.exec] : []

          content {
            command = exec.value.command
          }
        }

        dynamic "http_get" {
          for_each = custom_metrics.value.http_get != null ? [custom_metrics.value.http_get] : []
          content {
            path = http_get.value.path
            port = http_get.value.port
          }
        }
      }
    }

    dynamic "asset_model" {
      for_each = var.training_job_asset_model != null ? [var.training_job_asset_model] : []

      content {
        name    = asset_model.value.name
        version = asset_model.value.version
        type    = asset_model.value.type
        code    = asset_model.value.code
        desc    = asset_model.value.desc
        series  = asset_model.value.series
      }
    }

    dynamic "output_model" {
      for_each = var.training_job_output_model != null ? [var.training_job_output_model] : []

      content {
        obs {
          obs_path   = output_model.value.obs_path
          local_path = output_model.value.local_path
        }
      }
    }
  }

  tags = var.training_job_tags
}
```

**参数说明**：
- **kind**：训练作业类型，设置为"job"表示自定义训练作业
- **metadata.name**：训练作业的名称，通过引用输入变量training_job_name进行赋值
- **metadata.workspace_id**：工作空间ID，当workspace_id不为空时使用其值，否则引用创建的工作空间资源ID
- **metadata.annotations**：训练作业的注解，通过引用输入变量training_job_annotations进行赋值，默认值为空映射
- **metadata.description**：训练作业的描述，通过引用输入变量training_job_description进行赋值
- **algorithm.code_dir**：训练代码的OBS目录，通过引用输入变量training_job_code_dir进行赋值
- **algorithm.command**：容器启动命令，通过引用输入变量training_job_command进行赋值
- **algorithm.engine**：训练引擎配置，通过引用输入变量training_job_engine进行赋值，支持image_url、id、version、name等字段
- **algorithm.inputs**：训练输入配置，通过引用输入变量training_job_inputs进行赋值，支持数据集挂载
- **algorithm.environments**：环境变量，通过引用输入变量training_job_environments进行赋值，默认值为空映射
- **spec.resource.pool_id**：专属资源池ID，引用前面创建的专属资源池资源的ID
- **spec.resource.node_count**：资源副本数，通过引用输入变量resource_node_count进行赋值，默认值为1
- **spec.volumes**：卷挂载配置，通过引用输入变量training_job_volumes进行赋值，支持NFS、PFS和OBS挂载
- **spec.log_export_path**：日志导出路径，当training_job_log_export_path_obs_url不为空时配置OBS日志导出路径
- **spec.log_export_config**：日志导出配置，当training_job_log_export_config_version不为空时配置日志导出版本
- **spec.auto_stop**：自动停止配置，当training_job_auto_stop_duration不为0时配置自动停止时长，时间单位为HOURS
- **spec.notification**：通知配置，当training_job_notification_topic_urn或topic_name不为空时配置SMN通知
- **spec.custom_metrics**：自定义指标配置，通过引用输入变量custom_metrics进行赋值
- **spec.asset_model**：资产模型配置，通过引用输入变量training_job_asset_model进行赋值
- **spec.output_model**：输出模型配置，通过引用输入变量training_job_output_model进行赋值
- **tags**：训练作业标签，通过引用输入变量training_job_tags进行赋值，默认值为空映射

### 15. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`terraform.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC网络配置
vpc_name              = "tf_test_vpc"
subnet_name           = "tf_test_subnet"
security_group_name   = "tf_test_security_group"

# SFS Turbo配置
turbo_name = "tf_test_sfs_turbo"

# ModelArts网络与资源池配置
network_name           = "tf-test-network"
resource_pool_name     = "tf-test-resource-pool"
resource_pool_flavor_id = "your_resource_pool_flavor_id"

# 训练作业配置
training_job_name     = "tf_test_training_job"
training_job_code_dir = "your_training_code_dir"
training_job_command  = "your_training_command"

training_job_engine = {
  image_url = "your_swr_image_url"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="training_job_name=my-job"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 16. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建专属资源池及自定义训练作业
4. 运行 `terraform show` 查看已创建的专属资源池及自定义训练作业详情

## 参考信息

- [华为云ModelArts产品文档](https://support.huaweicloud.com/modelarts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ModelArts专属资源池自定义训练作业最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/modelarts/custom-training-job-with-dedicated-resource-pool)
