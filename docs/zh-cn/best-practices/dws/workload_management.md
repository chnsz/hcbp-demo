# 部署工作负载管理

## 应用场景

数据仓库服务（Data Warehouse Service，DWS）是华为云提供的在线联机分析处理（OLAP）企业级数据仓库服务，面向海量数据分析场景提供高性能、高可靠、易扩展的数据仓库能力。当多个业务或租户共享同一套 DWS 集群时，需要通过工作负载管理对 CPU、内存、并发等资源进行隔离与调度，避免个别业务抢占资源影响整体稳定性。

本最佳实践将介绍如何使用Terraform自动化部署DWS工作负载管理，包括VPC、子网与安全组创建，DWS集群创建，工作负载队列配置，集群用户创建，队列与用户关联，工作负载计划与计划阶段配置以及异常规则配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DWS集群规格（data.huaweicloud_dws_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dws_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [DWS集群（huaweicloud_dws_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_cluster)
- [DWS工作负载队列（huaweicloud_dws_workload_queue）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_workload_queue)
- [DWS集群用户（huaweicloud_dws_cluster_user）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_cluster_user)
- [DWS工作负载队列与用户关联（huaweicloud_dws_workload_queue_user_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_workload_queue_user_associate)
- [DWS工作负载计划（huaweicloud_dws_workload_plan）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_workload_plan)
- [DWS工作负载计划阶段（huaweicloud_dws_workload_plan_stage）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_workload_plan_stage)
- [DWS集群异常规则（huaweicloud_dws_cluster_exception_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_cluster_exception_rule)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dws_cluster

data.huaweicloud_dws_flavors
    └── huaweicloud_dws_cluster

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dws_cluster

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
    └── huaweicloud_dws_cluster

huaweicloud_dws_cluster
    └── huaweicloud_dws_workload_queue
    └── huaweicloud_dws_cluster_user
    └── huaweicloud_dws_workload_plan
    └── huaweicloud_dws_cluster_exception_rule

huaweicloud_dws_workload_queue
    └── huaweicloud_dws_workload_queue_user_associate
    └── huaweicloud_dws_workload_plan_stage

huaweicloud_dws_cluster_user
    └── huaweicloud_dws_workload_queue_user_associate

huaweicloud_dws_workload_plan
    └── huaweicloud_dws_workload_plan_stage
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询DWS集群可用的可用区：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
variable "availability_zone" {
  description = "The availability zone of the DWS cluster"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：当未指定可用区时，通过引用输入变量 availability_zone 判断是否需要查询可用区列表

### 3. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **name**：通过引用输入变量 vpc_name 进行赋值
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，为空时不指定企业项目

### 4. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：通过引用资源 huaweicloud_vpc.test 的ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，为空时基于VPC网段自动划分子网
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，为空时基于子网网段自动计算网关地址

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

variable "security_group_delete_default_rules" {
  description = "Whether to delete the default rules of the security group"
  type        = bool
  default     = true
}

resource "huaweicloud_networking_secgroup" "test" {
  name                  = var.security_group_name
  delete_default_rules  = var.security_group_delete_default_rules
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：通过引用输入变量 security_group_delete_default_rules 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，为空时不指定企业项目

### 6. 创建安全组规则

在TF文件（如main.tf）中添加以下脚本以创建允许DWS客户端访问的安全组规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组规则
variable "security_group_rule_ports" {
  description = "The security group ingress rule ports for DWS"
  type        = string
  default     = "8000-10000"
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = huaweicloud_vpc.test.cidr
  ports             = var.security_group_rule_ports
  protocol          = "tcp"
}
```

**参数说明**：
- **security_group_id**：通过引用资源 huaweicloud_networking_secgroup.test 的ID进行赋值
- **direction**：安全组规则方向，取值为 ingress
- **ethertype**：网络类型，取值为 IPv4
- **remote_ip_prefix**：通过引用资源 huaweicloud_vpc.test 的网段进行赋值
- **ports**：通过引用输入变量 security_group_rule_ports 进行赋值
- **protocol**：协议类型，取值为 tcp

### 7. 查询DWS集群规格

在TF文件（如main.tf）中添加以下脚本以查询DWS集群规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DWS集群规格
variable "cluster_node_type" {
  description = "The flavor of the DWS cluster node"
  type        = string
  default     = ""
  nullable    = false
}

variable "cluster_version" {
  description = "The version of the DWS cluster"
  type        = string
  default     = ""
  nullable    = false
}

variable "cluster_vcpus" {
  description = "The vcpus of the DWS cluster"
  type        = number
  default     = 4
}

variable "cluster_memory" {
  description = "The memory of the DWS cluster"
  type        = number
  default     = 32
}

variable "cluster_datastore_type" {
  description = "The datastore type of the DWS cluster"
  type        = string
  default     = "dws"
}

data "huaweicloud_dws_flavors" "test" {
  count = var.cluster_node_type == "" || var.cluster_version == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  vcpus             = var.cluster_vcpus
  memory            = var.cluster_memory
  datastore_type    = var.cluster_datastore_type
}
```

**参数说明**：
- **count**：当未指定节点规格或集群版本时，通过引用输入变量 cluster_node_type 与 cluster_version 判断是否需要查询集群规格
- **availability_zone**：通过引用输入变量 availability_zone 或数据源 huaweicloud_availability_zones.test 的可用区进行赋值
- **vcpus**：通过引用输入变量 cluster_vcpus 进行赋值
- **memory**：通过引用输入变量 cluster_memory 进行赋值
- **datastore_type**：通过引用输入变量 cluster_datastore_type 进行赋值

### 8. 创建DWS集群

在TF文件（如main.tf）中添加以下脚本以创建DWS集群：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DWS集群
variable "cluster_name" {
  description = "The name of the DWS cluster"
  type        = string
}

variable "cluster_number_of_node" {
  description = "The number of nodes in the DWS cluster"
  type        = number
  default     = 3
}

variable "cluster_number_of_cn" {
  description = "The number of CN nodes in the DWS cluster"
  type        = number
  default     = 3
}

variable "cluster_admin_user_name" {
  description = "The administrator username of the DWS cluster"
  type        = string
}

variable "cluster_admin_user_pwd" {
  description = "The administrator password of the DWS cluster"
  type        = string
  sensitive   = true
}

variable "cluster_volume_type" {
  description = "The volume type of the DWS cluster"
  type        = string
  default     = "SSD"
}

variable "cluster_volume_capacity" {
  description = "The volume capacity of the DWS cluster in GB"
  type        = string
  default     = "100"
}

resource "huaweicloud_dws_cluster" "test" {
  name                  = var.cluster_name
  node_type             = var.cluster_node_type != "" ? var.cluster_node_type : try(data.huaweicloud_dws_flavors.test[0].flavors[0].flavor_id, null)
  number_of_node        = var.cluster_number_of_node
  number_of_cn          = var.cluster_number_of_cn
  version               = var.cluster_version != "" ? var.cluster_version : try(data.huaweicloud_dws_flavors.test[0].flavors[0].datastore_version, null)
  vpc_id                = huaweicloud_vpc.test.id
  network_id            = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  user_name             = var.cluster_admin_user_name
  user_pwd              = var.cluster_admin_user_pwd
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  volume {
    type     = var.cluster_volume_type
    capacity = var.cluster_volume_capacity
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 cluster_name 进行赋值
- **node_type**：通过引用输入变量 cluster_node_type 或数据源 huaweicloud_dws_flavors.test 的规格ID进行赋值
- **number_of_node**：通过引用输入变量 cluster_number_of_node 进行赋值
- **number_of_cn**：通过引用输入变量 cluster_number_of_cn 进行赋值
- **version**：通过引用输入变量 cluster_version 或数据源 huaweicloud_dws_flavors.test 的版本进行赋值
- **vpc_id**：通过引用资源 huaweicloud_vpc.test 的ID进行赋值
- **network_id**：通过引用资源 huaweicloud_vpc_subnet.test 的ID进行赋值
- **security_group_id**：通过引用资源 huaweicloud_networking_secgroup.test 的ID进行赋值
- **availability_zone**：通过引用输入变量 availability_zone 或数据源 huaweicloud_availability_zones.test 的可用区进行赋值
- **user_name**：通过引用输入变量 cluster_admin_user_name 进行赋值
- **user_pwd**：通过引用输入变量 cluster_admin_user_pwd 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，为空时不指定企业项目
- **volume.type**：通过引用输入变量 cluster_volume_type 进行赋值
- **volume.capacity**：通过引用输入变量 cluster_volume_capacity 进行赋值

### 9. 创建工作负载队列

在TF文件（如main.tf）中添加以下脚本以创建工作负载队列：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建工作负载队列
variable "workload_queue_name" {
  description = "The name of the workload queue"
  type        = string
}

variable "workload_queue_configurations" {
  description = "The configurations of the workload queue"

  type = list(object({
    resource_name  = string
    resource_value = string
  }))
}

resource "huaweicloud_dws_workload_queue" "test" {
  cluster_id = huaweicloud_dws_cluster.test.id
  name       = var.workload_queue_name

  dynamic "configuration" {
    for_each = var.workload_queue_configurations

    content {
      resource_name  = configuration.value.resource_name
      resource_value = configuration.value.resource_value
    }
  }
}
```

**参数说明**：
- **cluster_id**：通过引用资源 huaweicloud_dws_cluster.test 的ID进行赋值
- **name**：通过引用输入变量 workload_queue_name 进行赋值
- **configuration.resource_name**：通过引用输入变量 workload_queue_configurations 中的 resource_name 进行赋值
- **configuration.resource_value**：通过引用输入变量 workload_queue_configurations 中的 resource_value 进行赋值

### 10. 创建集群用户

在TF文件（如main.tf）中添加以下脚本以创建集群用户：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建集群用户
variable "user_name" {
  description = "The name of the cluster user"
  type        = string
}

variable "user_password" {
  description = "The password of the cluster user"
  type        = string
  sensitive   = true
}

variable "user_description" {
  description = "The description of the cluster user"
  type        = string
  default     = ""
  nullable    = false
}

variable "user_cascade" {
  description = "Whether to cascade delete dependencies when deleting the user or role"
  type        = bool
  default     = true
}

variable "user_login" {
  description = "Whether to allow the user to log in"
  type        = bool
  default     = true
}

variable "user_create_role" {
  description = "Whether to grant the permission to create roles"
  type        = bool
  default     = true
}

variable "user_create_db" {
  description = "Whether to grant the permission to create databases"
  type        = bool
  default     = true
}

variable "user_system_admin" {
  description = "Whether to grant the system administrator permission"
  type        = bool
  default     = null
}

variable "user_audit_admin" {
  description = "Whether to grant the audit administrator permission"
  type        = bool
  default     = null
}

variable "user_inherit" {
  description = "Whether to inherit permissions from roles"
  type        = bool
  default     = true
}

variable "user_use_ft" {
  description = "Whether to grant the external table permission"
  type        = bool
  default     = null
}

variable "user_conn_limit" {
  description = "The maximum number of concurrent connections. -1 means unlimited"
  type        = number
  default     = -1
}

variable "user_replication" {
  description = "Whether to grant the replication permission"
  type        = bool
  default     = null
}

variable "user_valid_begin" {
  description = "The valid begin time of the cluster user"
  type        = string
  default     = ""
  nullable    = false
}

variable "user_valid_until" {
  description = "The valid until time of the cluster user"
  type        = string
  default     = ""
  nullable    = false
}

variable "user_grant_list" {
  description = "The set of grants for the cluster user"

  type = list(object({
    type                 = string
    database             = optional(string)
    schema_name          = optional(string)
    object_name          = optional(string)
    all_object           = optional(bool)
    future               = optional(bool)
    future_object_owners = optional(string)
    column_names         = optional(list(string))

    privileges = list(object({
      permission = string
      grant_with = bool
    }))
  }))

  default = []
}

resource "huaweicloud_dws_cluster_user" "test" {
  cluster_id   = huaweicloud_dws_cluster.test.id
  type         = "user"
  name         = var.user_name
  password     = var.user_password
  description  = var.user_description != "" ? var.user_description : null
  cascade      = var.user_cascade
  login        = var.user_login
  create_role  = var.user_create_role
  create_db    = var.user_create_db
  system_admin = var.user_system_admin
  audit_admin  = var.user_audit_admin
  inherit      = var.user_inherit
  use_ft       = var.user_use_ft
  conn_limit   = var.user_conn_limit
  replication  = var.user_replication
  valid_begin  = var.user_valid_begin
  valid_until  = var.user_valid_until

  dynamic "grant_list" {
    for_each = var.user_grant_list

    content {
      type                 = grant_list.value.type
      database             = grant_list.value.database
      schema_name          = grant_list.value.schema_name
      object_name          = grant_list.value.object_name
      all_object           = grant_list.value.all_object
      future               = grant_list.value.future
      future_object_owners = grant_list.value.future_object_owners
      column_names         = grant_list.value.column_names

      dynamic "privileges" {
        for_each = grant_list.value.privileges

        content {
          permission = privileges.value.permission
          grant_with = privileges.value.grant_with
        }
      }
    }
  }
}
```

**参数说明**：
- **cluster_id**：通过引用资源 huaweicloud_dws_cluster.test 的ID进行赋值
- **type**：用户类型，取值为 user
- **name**：通过引用输入变量 user_name 进行赋值
- **password**：通过引用输入变量 user_password 进行赋值
- **description**：通过引用输入变量 user_description 进行赋值，为空时不指定描述
- **cascade**：通过引用输入变量 user_cascade 进行赋值
- **login**：通过引用输入变量 user_login 进行赋值
- **create_role**：通过引用输入变量 user_create_role 进行赋值
- **create_db**：通过引用输入变量 user_create_db 进行赋值
- **system_admin**：通过引用输入变量 user_system_admin 进行赋值
- **audit_admin**：通过引用输入变量 user_audit_admin 进行赋值
- **inherit**：通过引用输入变量 user_inherit 进行赋值
- **use_ft**：通过引用输入变量 user_use_ft 进行赋值
- **conn_limit**：通过引用输入变量 user_conn_limit 进行赋值
- **replication**：通过引用输入变量 user_replication 进行赋值
- **valid_begin**：通过引用输入变量 user_valid_begin 进行赋值
- **valid_until**：通过引用输入变量 user_valid_until 进行赋值
- **grant_list**：通过引用输入变量 user_grant_list 进行赋值，用于配置用户的授权信息

### 11. 关联工作负载队列与用户

在TF文件（如main.tf）中添加以下脚本以关联工作负载队列与用户：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下关联工作负载队列与用户
resource "huaweicloud_dws_workload_queue_user_associate" "test" {
  cluster_id = huaweicloud_dws_cluster.test.id
  queue_name = huaweicloud_dws_workload_queue.test.name
  user_names = [huaweicloud_dws_cluster_user.test.name]
}
```

**参数说明**：
- **cluster_id**：通过引用资源 huaweicloud_dws_cluster.test 的ID进行赋值
- **queue_name**：通过引用资源 huaweicloud_dws_workload_queue.test 的名称进行赋值
- **user_names**：通过引用资源 huaweicloud_dws_cluster_user.test 的名称进行赋值

### 12. 创建工作负载计划

在TF文件（如main.tf）中添加以下脚本以创建工作负载计划：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建工作负载计划
variable "workload_plan_name" {
  description = "The name of the workload plan"
  type        = string
}

resource "huaweicloud_dws_workload_plan" "test" {
  cluster_id = huaweicloud_dws_cluster.test.id
  name       = var.workload_plan_name
}
```

**参数说明**：
- **cluster_id**：通过引用资源 huaweicloud_dws_cluster.test 的ID进行赋值
- **name**：通过引用输入变量 workload_plan_name 进行赋值

### 13. 创建工作负载计划阶段

在TF文件（如main.tf）中添加以下脚本以创建工作负载计划阶段：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建工作负载计划阶段
variable "workload_plan_stage_name" {
  description = "The name of the workload plan stage"
  type        = string
}

variable "workload_plan_stage_month" {
  description = "The month of the workload plan stage"
  type        = string
  default     = null
}

variable "workload_plan_stage_day" {
  description = "The day of the workload plan stage"
  type        = string
  default     = null
}

variable "workload_plan_stage_start_time" {
  description = "The start time of the workload plan stage. The format is hh:mm:ss"
  type        = string
  default     = "00:00:00"
}

variable "workload_plan_stage_end_time" {
  description = "The end time of the workload plan stage. The format is hh:mm:ss"
  type        = string
  default     = "23:59:59"
}

variable "workload_plan_stage_configurations" {
  description = "The configurations of the workload plan stage"

  type = list(object({
    resource_name        = string
    resource_value       = string
    value_unit           = optional(string)
    resource_description = optional(string)
  }))
}

resource "huaweicloud_dws_workload_plan_stage" "test" {
  cluster_id = huaweicloud_dws_cluster.test.id
  plan_id    = huaweicloud_dws_workload_plan.test.id
  name       = var.workload_plan_stage_name
  month      = var.workload_plan_stage_month
  day        = var.workload_plan_stage_day
  start_time = var.workload_plan_stage_start_time
  end_time   = var.workload_plan_stage_end_time

  queues {
    name = huaweicloud_dws_workload_queue.test.name

    dynamic "configuration" {
      for_each = var.workload_plan_stage_configurations

      content {
        resource_name        = configuration.value.resource_name
        resource_value       = configuration.value.resource_value
        value_unit           = configuration.value.value_unit
        resource_description = configuration.value.resource_description
      }
    }
  }
}
```

**参数说明**：
- **cluster_id**：通过引用资源 huaweicloud_dws_cluster.test 的ID进行赋值
- **plan_id**：通过引用资源 huaweicloud_dws_workload_plan.test 的ID进行赋值
- **name**：通过引用输入变量 workload_plan_stage_name 进行赋值
- **month**：通过引用输入变量 workload_plan_stage_month 进行赋值，未指定时应用于所有月份
- **day**：通过引用输入变量 workload_plan_stage_day 进行赋值，未指定时应用于所有日期
- **start_time**：通过引用输入变量 workload_plan_stage_start_time 进行赋值
- **end_time**：通过引用输入变量 workload_plan_stage_end_time 进行赋值
- **queues.name**：通过引用资源 huaweicloud_dws_workload_queue.test 的名称进行赋值
- **configuration.resource_name**：通过引用输入变量 workload_plan_stage_configurations 中的 resource_name 进行赋值
- **configuration.resource_value**：通过引用输入变量 workload_plan_stage_configurations 中的 resource_value 进行赋值
- **configuration.value_unit**：通过引用输入变量 workload_plan_stage_configurations 中的 value_unit 进行赋值
- **configuration.resource_description**：通过引用输入变量 workload_plan_stage_configurations 中的 resource_description 进行赋值

### 14. 创建集群异常规则

在TF文件（如main.tf）中添加以下脚本以创建集群异常规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建集群异常规则
variable "exception_rule_name" {
  description = "The name of the cluster exception rule"
  type        = string
}

variable "exception_rule_configurations" {
  description = "The configurations of the exception rule"

  type = list(object({
    key   = string
    value = string
  }))
}

resource "huaweicloud_dws_cluster_exception_rule" "test" {
  cluster_id = huaweicloud_dws_cluster.test.id
  name       = var.exception_rule_name

  dynamic "configurations" {
    for_each = var.exception_rule_configurations

    content {
      key   = configurations.value.key
      value = configurations.value.value
    }
  }
}
```

**参数说明**：
- **cluster_id**：通过引用资源 huaweicloud_dws_cluster.test 的ID进行赋值
- **name**：通过引用输入变量 exception_rule_name 进行赋值
- **configurations.key**：通过引用输入变量 exception_rule_configurations 中的 key 进行赋值
- **configurations.value**：通过引用输入变量 exception_rule_configurations 中的 value 进行赋值

### 15. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name                = "tf_test_dws_vpc"
vpc_cidr                = "192.168.0.0/16"
subnet_name             = "tf_test_dws_subnet"
security_group_name     = "tf_test_dws_sg"
cluster_name            = "tf_test_cluster"
cluster_admin_user_name = "dbadmin"
cluster_admin_user_pwd  = "YourPassword@123"
workload_queue_name     = "tf_test_queue"

workload_queue_configurations = [
  {
    resource_name  = "cpu_limit"
    resource_value = "10"
  },
  {
    resource_name  = "memory"
    resource_value = "10"
  },
  {
    resource_name  = "tablespace"
    resource_value = "-1"
  },
  {
    resource_name  = "activestatements"
    resource_value = "-1"
  }
]

user_name                = "tf_test_user"
user_password            = "YourPassword@456"
workload_plan_name       = "tf_test_plan"
workload_plan_stage_name = "tf_test_stage"

workload_plan_stage_configurations = [
  {
    resource_name  = "cpu"
    resource_value = "1"
  },
  {
    resource_name  = "cpu_limit"
    resource_value = "0"
  },
  {
    resource_name  = "memory"
    resource_value = "10"
  },
  {
    resource_name  = "concurrency"
    resource_value = "10"
  },
  {
    resource_name  = "shortQueryConcurrencyNum"
    resource_value = "-1"
  }
]

exception_rule_name = "tf_test_exception_rule"

exception_rule_configurations = [
  {
    key   = "action"
    value = "abort"
  },
  {
    key   = "blocktime"
    value = "300"
  },
  {
    key   = "elapsedtime"
    value = "400"
  },
  {
    key   = "allcputime"
    value = "500"
  },
  {
    key   = "cpuskewpercent"
    value = "60"
  },
  {
    key   = "cpuavgpercent"
    value = "70"
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 16. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建工作负载管理资源
4. 运行 `terraform show` 查看已创建的工作负载管理资源

## 参考信息

- [华为云数据仓库服务产品文档](https://support.huaweicloud.com/dws/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DWS工作负载管理最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dws/workload-management)
