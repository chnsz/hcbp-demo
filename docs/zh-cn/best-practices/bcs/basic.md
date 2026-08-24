# 部署基础实例

## 应用场景

区块链服务（Blockchain Service，简称BCS）是面向企业及开发者提供的区块链技术服务平台，可以帮助您快速部署、管理、维护区块链网络，降低使用区块链的门槛，让您专注于自身业务的开发与创新，实现业务快速上链。BCS支持创建Hyperledger Fabric增强版和华为云区块链引擎实例，提供用户管理、节点管理、运维监控等能力，满足企业级和金融级业务要求。

本最佳实践将介绍如何使用Terraform自动化部署一个基础的BCS实例，包括指定服务版本、Fabric版本、共识算法、关联CCE集群，以及配置区块生成参数、SFS Turbo、Peer组织和通道。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [BCS实例资源（huaweicloud_bcs_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/bcs_instance)

### 资源/数据源依赖关系

```
huaweicloud_bcs_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建BCS实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建BCS实例资源：

```hcl
variable "instance_name" {
  description = "The unique name of the BCS instance"
  type        = string
}

variable "edition" {
  description = "The service edition of the BCS instance"
  type        = number
}

variable "fabric_version" {
  description = "The version of fabric for the BCS instance"
  type        = string
}

variable "consensus" {
  description = "The consensus algorithm used by the BCS instance"
  type        = string
}

variable "orderer_node_num" {
  description = "The number of peers in the orderer organization"
  type        = number
  default     = 1
}

variable "cce_cluster_id" {
  description = "The ID of the CCE cluster to attach to the BCS instance"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project that the BCS instance belongs to"
  type        = string
}

variable "instance_password" {
  description = "The resource access and blockchain management password"
  type        = string
  sensitive   = true
}

variable "volume_type" {
  description = "The storage volume type to attach to each organization of the BCS instance"
  type        = string
  default     = "nfs"
}

variable "org_disk_size" {
  description = "The storage capacity of peer organization"
  type        = number
  default     = 100
}

variable "block_info" {
  description = "The configuration of block generation"
  type = list(object({
    generation_interval  = optional(number, 2)
    transaction_quantity = optional(number, 500)
    block_size           = optional(number, 2)
  }))

  default = []
}

variable "sfs_turbo" {
  description = "The SFS Turbo configuration for BCS instance"
  type = list(object({
    share_type        = optional(string, "STANDARD")
    type              = optional(string, "efs-ha")
    availability_zone = optional(string, "")
    flavor            = optional(string, "sfs.turbo.20MBps")
  }))

  default = []

  validation {
    condition     = var.volume_type != "efs" || var.edition != 4 || length(var.sfs_turbo) > 0
    error_message = "When using \"volume_type = \"efs\"\" with \"edition = 4\", you must configure \"sfs_turbo\"."
  }
}

variable "peer_orgs" {
  description = "The array of one or more peer organizations to attach to the BCS instance"
  type = list(object({
    org_name = string
    count    = number
  }))

  default = [
    {
      org_name = "organization"
      count    = 2
    }
  ]
}

variable "channels" {
  description = "The array of one or more channels to attach to the BCS instance"
  type = list(object({
    name      = string
    org_names = list(string)
  }))

  default = [
    {
      name      = "channel"
      org_names = ["organization"]
    }
  ]
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建BCS实例资源
resource "huaweicloud_bcs_instance" "test" {
  name                  = var.instance_name
  edition               = var.edition
  fabric_version        = var.fabric_version
  consensus             = var.consensus
  orderer_node_num      = var.orderer_node_num
  cce_cluster_id        = var.cce_cluster_id
  enterprise_project_id = var.enterprise_project_id
  password              = var.instance_password
  volume_type           = var.volume_type
  org_disk_size         = var.org_disk_size

  dynamic "block_info" {
    for_each = var.block_info

    content {
      generation_interval  = block_info.value.generation_interval
      transaction_quantity = block_info.value.transaction_quantity
      block_size           = block_info.value.block_size
    }
  }

  dynamic "sfs_turbo" {
    for_each = var.sfs_turbo

    content {
      share_type        = sfs_turbo.value.share_type
      type              = sfs_turbo.value.type
      availability_zone = sfs_turbo.value.availability_zone
      flavor            = sfs_turbo.value.flavor
    }
  }

  dynamic "peer_orgs" {
    for_each = var.peer_orgs

    content {
      org_name = peer_orgs.value.org_name
      count    = peer_orgs.value.count
    }
  }

  dynamic "channels" {
    for_each = var.channels

    content {
      name      = channels.value.name
      org_names = channels.value.org_names
    }
  }
}
```

**参数说明**：
- **name**：BCS实例的唯一名称，通过引用输入变量instance_name进行赋值
- **edition**：BCS实例的服务版本，通过引用输入变量edition进行赋值，有效值为1、2、4
- **fabric_version**：BCS实例使用的Fabric版本，通过引用输入变量fabric_version进行赋值
- **consensus**：BCS实例使用的共识算法，通过引用输入变量consensus进行赋值
- **orderer_node_num**：Orderer组织中的节点数量，通过引用输入变量orderer_node_num进行赋值，默认为1
- **cce_cluster_id**：关联到BCS实例的CCE集群ID，通过引用输入变量cce_cluster_id进行赋值
- **enterprise_project_id**：BCS实例所属企业项目的ID，通过引用输入变量enterprise_project_id进行赋值
- **password**：资源访问和区块链管理密码，通过引用输入变量instance_password进行赋值
- **volume_type**：挂载到各组织的存储卷类型，通过引用输入变量volume_type进行赋值，默认为"nfs"，有效值为"nfs"（SFS）和"efs"（SFS Turbo）
- **org_disk_size**：Peer组织的存储容量，通过引用输入变量org_disk_size进行赋值，默认为100
- **block_info**：区块生成配置，通过动态块 `dynamic "block_info"` 根据输入变量block_info创建
  - **generation_interval**：出块时间间隔（单位：秒），默认为2
  - **transaction_quantity**：区块中包含的交易数量，默认为500
  - **block_size**：区块大小（单位：MB），默认为2
- **sfs_turbo**：SFS Turbo文件系统配置，通过动态块 `dynamic "sfs_turbo"` 根据输入变量sfs_turbo创建
  - **share_type**：SFS Turbo共享类型，默认为"STANDARD"
  - **type**：SFS Turbo类型，默认为"efs-ha"
  - **availability_zone**：SFS Turbo所在可用区，默认为空字符串
  - **flavor**：SFS Turbo规格，默认为"sfs.turbo.20MBps"
- **peer_orgs**：挂载到BCS实例的Peer组织列表，通过动态块 `dynamic "peer_orgs"` 根据输入变量peer_orgs创建，默认创建一个名为organization且节点数为2的组织
  - **org_name**：Peer组织名称
  - **count**：组织中的Peer节点数量
- **channels**：挂载到BCS实例的通道列表，通过动态块 `dynamic "channels"` 根据输入变量channels创建，默认创建一个名为channel的通道
  - **name**：通道名称
  - **org_names**：通道关联的Peer组织名称列表

> 注意：BCS服务需要独占CCE集群，部署前请确保目标CCE集群未被其他BCS实例占用。当`volume_type`为"efs"且`edition`为4时，必须配置`sfs_turbo`。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# BCS实例配置
instance_name         = "basic-bcs-instance"
edition               = 4
fabric_version        = "4.0.35"
consensus             = "etcdraft"
orderer_node_num      = 3
cce_cluster_id        = "your-cce-cluster-id"
enterprise_project_id = "your-enterprise-project-id"
instance_password     = "your-instance-password"
volume_type           = "efs"
org_disk_size         = 3686

block_info = [
  {
    generation_interval  = 2
    transaction_quantity = 500
    block_size           = 2
  }
]

# SFS Turbo配置（volume_type为efs且edition为4时必填）
sfs_turbo = [
  {
    share_type        = "STANDARD"
    type              = "efs-ha"
    availability_zone = "cn-north-4a"
    flavor            = "sfs.turbo.20MBps"
  }
]

# Peer组织配置
peer_orgs = [
  {
    org_name = "organization"
    count    = 2
  }
]

# 通道配置
channels = [
  {
    name      = "channel"
    org_names = ["organization"]
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="instance_name=basic-bcs-instance" -var="edition=4"`
2. 环境变量：`export TF_VAR_instance_name=basic-bcs-instance`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建BCS实例
4. 运行 `terraform show` 查看已创建的BCS实例

## 参考信息

- [华为云BCS产品文档](https://support.huaweicloud.com/bcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [BCS基础实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/bcs/basic)
