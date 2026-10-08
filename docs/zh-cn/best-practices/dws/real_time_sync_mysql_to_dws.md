# 部署MySQL到DWS的实时数据同步

## 应用场景

数据仓库服务（Data Warehouse Service，DWS）是华为云提供的在线联机分析处理（OLAP）企业级数据仓库服务，面向海量数据分析场景提供高性能、高可靠、易扩展的数据仓库能力。在实际业务中，源端业务数据往往存放在关系型数据库服务（RDS）MySQL 实例中，需要近实时地同步到 DWS 集群进行汇总分析，从而支撑报表、看板与商业智能（BI）等场景。

本最佳实践将介绍如何使用Terraform自动化部署一套 MySQL 到 DWS 的实时数据同步基础设施，包括共享 VPC、子网与安全组创建，RDS MySQL 实例创建，DWS 集群创建，DLI 弹性资源池与通用队列创建，以及增强型数据源连接创建并关联弹性资源池，为后续通过 DLI Flink OpenSource SQL 作业实现 MySQL CDC 到 DWS 的实时同步奠定基础。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS规格列表（data.huaweicloud_rds_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)
- [DWS规格列表（data.huaweicloud_dws_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dws_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RDS MySQL实例（huaweicloud_rds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DWS集群（huaweicloud_dws_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_cluster)
- [DLI弹性资源池（huaweicloud_dli_elastic_resource_pool）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_elastic_resource_pool)
- [DLI队列（huaweicloud_dli_queue）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_queue)
- [DLI增强型数据源连接（huaweicloud_dli_datasource_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_datasource_connection)
- [DLI增强型数据源连接关联（huaweicloud_dli_datasource_connection_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_datasource_connection_associate)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_rds_flavors
    │       └── huaweicloud_rds_instance
    └── data.huaweicloud_dws_flavors
            └── huaweicloud_dws_cluster

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            ├── huaweicloud_rds_instance
            ├── huaweicloud_dws_cluster
            └── huaweicloud_dli_datasource_connection

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    ├── huaweicloud_rds_instance
    └── huaweicloud_dws_cluster

huaweicloud_dli_elastic_resource_pool
    ├── huaweicloud_dli_queue
    └── huaweicloud_dli_datasource_connection_associate

huaweicloud_dli_datasource_connection
    └── huaweicloud_dli_datasource_connection_associate
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询当前区域下的可用区列表，用于后续创建 RDS 实例与 DWS 集群时指定可用区：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
variable "availability_zone" {
  description = "The availability zone. If empty, the first available zone is used"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：

- **count**：当输入变量 `availability_zone` 为空时创建该数据源，用于自动获取当前区域下的可用区列表

### 3. 创建虚拟私有云

在TF文件中添加以下脚本以创建 RDS 与 DWS 共享的虚拟私有云：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云
variable "vpc_name" {
  description = "The name of the VPC shared by RDS and DWS"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC. Must differ from the DLI elastic resource pool CIDR"
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

- **name**：通过引用输入变量 `vpc_name` 进行赋值，用于指定虚拟私有云的名称
- **cidr**：通过引用输入变量 `vpc_cidr` 进行赋值，用于指定虚拟私有云的网段，需与 DLI 弹性资源池网段不同
- **enterprise_project_id**：通过引用输入变量 `enterprise_project_id` 进行赋值，用于指定企业项目 ID

### 4. 创建虚拟私有云子网

在TF文件中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet. If empty, it is calculated from the VPC CIDR"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet. If empty, it is calculated from the subnet CIDR"
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

- **vpc_id**：通过引用 `huaweicloud_vpc.test.id` 进行赋值，用于指定子网所属的虚拟私有云
- **name**：通过引用输入变量 `subnet_name` 进行赋值，用于指定子网的名称
- **cidr**：通过引用输入变量 `subnet_cidr` 进行赋值，若为空则基于 VPC 网段自动计算
- **gateway_ip**：通过引用输入变量 `subnet_gateway_ip` 进行赋值，若为空则基于子网网段自动计算

### 5. 创建安全组及其规则

在TF文件中添加以下脚本以创建安全组，并放通 DLI 弹性资源池网段访问 DWS 与 RDS 端口：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组及其规则
variable "security_group_name" {
  description = "The name of the security group shared by the RDS instance and DWS cluster"
  type        = string
}

variable "security_group_delete_default_rules" {
  description = "Whether to delete the default rules of the security group"
  type        = bool
  default     = true
}

variable "dws_port" {
  description = "The service port of the DWS cluster"
  type        = number
  default     = 8000
}

variable "rds_db_port" {
  description = "The database port of the RDS MySQL instance"
  type        = number
  default     = 3306
}

variable "elastic_resource_pool_cidr" {
  description = "The CIDR block of the DLI elastic resource pool. Must differ from the VPC CIDR"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                  = var.security_group_name
  delete_default_rules  = var.security_group_delete_default_rules
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}

# DWS and RDS ports are opened to DLI elastic resource pool
resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  ports             = "${var.dws_port},${var.rds_db_port}"
  remote_ip_prefix  = var.elastic_resource_pool_cidr
}
```

**参数说明**：

- **name**：通过引用输入变量 `security_group_name` 进行赋值，用于指定安全组的名称
- **delete_default_rules**：通过引用输入变量 `security_group_delete_default_rules` 进行赋值，用于指定是否删除安全组默认规则
- **security_group_id**：通过引用 `huaweicloud_networking_secgroup.test.id` 进行赋值，用于指定安全组规则的所属安全组
- **direction**：设置为 `ingress`，表示入方向规则
- **protocol**：设置为 `tcp`，表示 TCP 协议
- **ports**：通过引用输入变量 `dws_port` 与 `rds_db_port` 进行赋值，用于放通 DWS 与 RDS 的服务端口
- **remote_ip_prefix**：通过引用输入变量 `elastic_resource_pool_cidr` 进行赋值，用于指定 DLI 弹性资源池网段

### 6. 查询RDS规格列表

在TF文件中添加以下脚本以查询 RDS MySQL 实例规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询RDS规格列表
variable "rds_flavor_id" {
  description = "The flavor ID of the RDS instance. If empty, it is queried from huaweicloud_rds_flavors"
  type        = string
  default     = ""
  nullable    = false
}

variable "rds_db_version" {
  description = "The MySQL version of the RDS instance"
  type        = string
  default     = "5.7"
}

variable "rds_instance_mode" {
  description = "The instance mode used to query RDS flavors"
  type        = string
  default     = "single"
}

variable "rds_flavor_vcpus" {
  description = "The vCPUs used to query RDS flavors"
  type        = number
  default     = 2
}

data "huaweicloud_rds_flavors" "test" {
  count = var.rds_flavor_id == "" ? 1 : 0

  db_type           = "MySQL"
  db_version        = var.rds_db_version
  instance_mode     = var.rds_instance_mode
  vcpus             = var.rds_flavor_vcpus
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**参数说明**：

- **count**：当输入变量 `rds_flavor_id` 为空时创建该数据源，用于自动查询 RDS 规格
- **db_type**：设置为 `MySQL`，表示查询 MySQL 类型规格
- **db_version**：通过引用输入变量 `rds_db_version` 进行赋值，用于指定 MySQL 版本
- **instance_mode**：通过引用输入变量 `rds_instance_mode` 进行赋值，用于指定实例模式
- **vcpus**：通过引用输入变量 `rds_flavor_vcpus` 进行赋值，用于指定规格的 vCPU 数量
- **availability_zone**：通过引用输入变量 `availability_zone` 或可用区列表数据源进行赋值

### 7. 创建RDS MySQL实例

在TF文件中添加以下脚本以创建 RDS MySQL 实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS MySQL实例
variable "rds_instance_name" {
  description = "The name of the RDS MySQL instance"
  type        = string
}

variable "rds_db_password" {
  description = "The root password of the RDS MySQL instance"
  type        = string
  sensitive   = true
}

variable "rds_volume_type" {
  description = "The volume type of the RDS instance"
  type        = string
  default     = "CLOUDSSD"
}

variable "rds_volume_size" {
  description = "The volume size of the RDS instance in GB"
  type        = number
  default     = 40
}

resource "huaweicloud_rds_instance" "test" {
  name              = var.rds_instance_name
  flavor            = var.rds_flavor_id != "" ? var.rds_flavor_id : try(data.huaweicloud_rds_flavors.test[0].flavors[0].name, null)
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  availability_zone = var.availability_zone != "" ? [var.availability_zone] : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null)

  db {
    type     = "MySQL"
    version  = var.rds_db_version
    port     = var.rds_db_port
    password = var.rds_db_password
  }

  volume {
    type = var.rds_volume_type
    size = var.rds_volume_size
  }

  lifecycle {
    ignore_changes = [flavor]
  }
}
```

**参数说明**：

- **name**：通过引用输入变量 `rds_instance_name` 进行赋值，用于指定 RDS 实例的名称
- **flavor**：通过引用输入变量 `rds_flavor_id` 或 RDS 规格列表数据源进行赋值
- **vpc_id**：通过引用 `huaweicloud_vpc.test.id` 进行赋值，用于指定实例所属的虚拟私有云
- **subnet_id**：通过引用 `huaweicloud_vpc_subnet.test.id` 进行赋值，用于指定实例所属的子网
- **security_group_id**：通过引用 `huaweicloud_networking_secgroup.test.id` 进行赋值，用于指定实例所属的安全组
- **availability_zone**：通过引用输入变量 `availability_zone` 或可用区列表数据源进行赋值
- **db.type**：设置为 `MySQL`，表示数据库类型
- **db.version**：通过引用输入变量 `rds_db_version` 进行赋值，用于指定数据库版本
- **db.port**：通过引用输入变量 `rds_db_port` 进行赋值，用于指定数据库端口
- **db.password**：通过引用输入变量 `rds_db_password` 进行赋值，用于指定数据库 root 用户密码
- **volume.type**：通过引用输入变量 `rds_volume_type` 进行赋值，用于指定存储类型
- **volume.size**：通过引用输入变量 `rds_volume_size` 进行赋值，用于指定存储大小

### 8. 查询DWS规格列表

在TF文件中添加以下脚本以查询 DWS 集群规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DWS规格列表
variable "dws_node_type" {
  description = "The flavor of the DWS cluster node. If empty, it is queried from huaweicloud_dws_flavors"
  type        = string
  default     = ""
  nullable    = false
}

variable "dws_version" {
  description = "The version of the DWS cluster. If empty, it is queried from huaweicloud_dws_flavors"
  type        = string
  default     = ""
  nullable    = false
}

variable "dws_flavor_vcpus" {
  description = "The vCPUs used to query DWS flavors"
  type        = number
  default     = 4
}

variable "dws_flavor_memory" {
  description = "The memory used to query DWS flavors"
  type        = number
  default     = 32
}

variable "dws_datastore_type" {
  description = "The datastore type of the DWS cluster"
  type        = string
  default     = "dws"
}

data "huaweicloud_dws_flavors" "test" {
  count = var.dws_node_type == "" || var.dws_version == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  vcpus             = var.dws_flavor_vcpus
  memory            = var.dws_flavor_memory
  datastore_type    = var.dws_datastore_type
}
```

**参数说明**：

- **count**：当输入变量 `dws_node_type` 或 `dws_version` 为空时创建该数据源，用于自动查询 DWS 规格
- **availability_zone**：通过引用输入变量 `availability_zone` 或可用区列表数据源进行赋值
- **vcpus**：通过引用输入变量 `dws_flavor_vcpus` 进行赋值，用于指定规格的 vCPU 数量
- **memory**：通过引用输入变量 `dws_flavor_memory` 进行赋值，用于指定规格的内存大小
- **datastore_type**：通过引用输入变量 `dws_datastore_type` 进行赋值，用于指定数据存储类型

### 9. 创建DWS集群

在TF文件中添加以下脚本以创建 DWS 集群：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DWS集群
variable "dws_cluster_name" {
  description = "The name of the DWS cluster"
  type        = string
}

variable "dws_number_of_node" {
  description = "The number of nodes in the DWS cluster"
  type        = number
  default     = 3
}

variable "dws_number_of_cn" {
  description = "The number of CN nodes in the DWS cluster"
  type        = number
  default     = 3
}

variable "dws_admin_user_name" {
  description = "The administrator username of the DWS cluster"
  type        = string
  default     = "dbadmin"
}

variable "dws_admin_user_pwd" {
  description = "The administrator password of the DWS cluster"
  type        = string
  sensitive   = true
}

variable "dws_volume_type" {
  description = "The volume type of the DWS cluster"
  type        = string
  default     = "SSD"
}

variable "dws_volume_capacity" {
  description = "The volume capacity of the DWS cluster in GB"
  type        = string
  default     = "100"
}

resource "huaweicloud_dws_cluster" "test" {
  name                  = var.dws_cluster_name
  node_type             = var.dws_node_type != "" ? var.dws_node_type : try(data.huaweicloud_dws_flavors.test[0].flavors[0].flavor_id, null)
  number_of_node        = var.dws_number_of_node
  number_of_cn          = var.dws_number_of_cn
  version               = var.dws_version != "" ? var.dws_version : try(data.huaweicloud_dws_flavors.test[0].flavors[0].datastore_version, null)
  vpc_id                = huaweicloud_vpc.test.id
  network_id            = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  user_name             = var.dws_admin_user_name
  user_pwd              = var.dws_admin_user_pwd
  port                  = var.dws_port
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  volume {
    type     = var.dws_volume_type
    capacity = var.dws_volume_capacity
  }
}
```

**参数说明**：

- **name**：通过引用输入变量 `dws_cluster_name` 进行赋值，用于指定 DWS 集群的名称
- **node_type**：通过引用输入变量 `dws_node_type` 或 DWS 规格列表数据源进行赋值
- **number_of_node**：通过引用输入变量 `dws_number_of_node` 进行赋值，用于指定集群节点数量
- **number_of_cn**：通过引用输入变量 `dws_number_of_cn` 进行赋值，用于指定 CN 节点数量
- **version**：通过引用输入变量 `dws_version` 或 DWS 规格列表数据源进行赋值
- **vpc_id**：通过引用 `huaweicloud_vpc.test.id` 进行赋值，用于指定集群所属的虚拟私有云
- **network_id**：通过引用 `huaweicloud_vpc_subnet.test.id` 进行赋值，用于指定集群所属的子网
- **security_group_id**：通过引用 `huaweicloud_networking_secgroup.test.id` 进行赋值，用于指定集群所属的安全组
- **availability_zone**：通过引用输入变量 `availability_zone` 或可用区列表数据源进行赋值
- **user_name**：通过引用输入变量 `dws_admin_user_name` 进行赋值，用于指定集群管理员用户名
- **user_pwd**：通过引用输入变量 `dws_admin_user_pwd` 进行赋值，用于指定集群管理员密码
- **port**：通过引用输入变量 `dws_port` 进行赋值，用于指定集群服务端口
- **volume.type**：通过引用输入变量 `dws_volume_type` 进行赋值，用于指定存储类型
- **volume.capacity**：通过引用输入变量 `dws_volume_capacity` 进行赋值，用于指定存储容量

### 10. 创建DLI弹性资源池

在TF文件中添加以下脚本以创建 DLI 弹性资源池：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DLI弹性资源池
variable "elastic_resource_pool_name" {
  description = "The name of the DLI elastic resource pool"
  type        = string
}

variable "elastic_resource_pool_description" {
  description = "The description of the DLI elastic resource pool"
  type        = string
  default     = ""
}

variable "elastic_resource_pool_min_cu" {
  description = "The minimum number of CUs for the DLI elastic resource pool"
  type        = number
  default     = 16
}

variable "elastic_resource_pool_max_cu" {
  description = "The maximum number of CUs for the DLI elastic resource pool"
  type        = number
  default     = 64
}

variable "elastic_resource_pool_label" {
  description = "The label of the DLI elastic resource pool"
  type        = map(string)

  default = {
    spec = "basic"
  }
}

resource "huaweicloud_dli_elastic_resource_pool" "test" {
  name                  = var.elastic_resource_pool_name
  description           = var.elastic_resource_pool_description
  min_cu                = var.elastic_resource_pool_min_cu
  max_cu                = var.elastic_resource_pool_max_cu
  cidr                  = var.elastic_resource_pool_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  label                 = var.elastic_resource_pool_label
}
```

**参数说明**：

- **name**：通过引用输入变量 `elastic_resource_pool_name` 进行赋值，用于指定弹性资源池的名称
- **description**：通过引用输入变量 `elastic_resource_pool_description` 进行赋值，用于指定弹性资源池的描述
- **min_cu**：通过引用输入变量 `elastic_resource_pool_min_cu` 进行赋值，用于指定弹性资源池的最小 CU 数
- **max_cu**：通过引用输入变量 `elastic_resource_pool_max_cu` 进行赋值，用于指定弹性资源池的最大 CU 数
- **cidr**：通过引用输入变量 `elastic_resource_pool_cidr` 进行赋值，用于指定弹性资源池的网段，需与 VPC 网段不同
- **label**：通过引用输入变量 `elastic_resource_pool_label` 进行赋值，用于指定弹性资源池的标签

### 11. 创建DLI队列

在TF文件中添加以下脚本以创建 DLI 通用队列：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DLI队列
variable "queue_name" {
  description = "The name of the DLI general queue"
  type        = string
}

variable "queue_cu_count" {
  description = "The CU count of the DLI queue"
  type        = number
  default     = 16
}

variable "queue_description" {
  description = "The description of the DLI queue"
  type        = string
  default     = ""
}

resource "huaweicloud_dli_queue" "test" {
  elastic_resource_pool_name = huaweicloud_dli_elastic_resource_pool.test.name
  resource_mode              = 1
  name                       = var.queue_name
  queue_type                 = "general"
  cu_count                   = var.queue_cu_count
  description                = var.queue_description
  enterprise_project_id      = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：

- **elastic_resource_pool_name**：通过引用 `huaweicloud_dli_elastic_resource_pool.test.name` 进行赋值，用于指定队列所属的弹性资源池
- **resource_mode**：设置为 `1`，表示队列使用弹性资源池模式
- **name**：通过引用输入变量 `queue_name` 进行赋值，用于指定队列的名称
- **queue_type**：设置为 `general`，表示通用队列
- **cu_count**：通过引用输入变量 `queue_cu_count` 进行赋值，用于指定队列的 CU 数
- **description**：通过引用输入变量 `queue_description` 进行赋值，用于指定队列的描述

### 12. 创建DLI增强型数据源连接并关联弹性资源池

在TF文件中添加以下脚本以创建 DLI 增强型数据源连接，并将其关联到弹性资源池：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DLI增强型数据源连接并关联弹性资源池
variable "datasource_connection_name" {
  description = "The name of the DLI enhanced datasource connection"
  type        = string
}

resource "huaweicloud_dli_datasource_connection" "test" {
  name      = var.datasource_connection_name
  vpc_id    = huaweicloud_vpc.test.id
  subnet_id = huaweicloud_vpc_subnet.test.id
}

resource "huaweicloud_dli_datasource_connection_associate" "test" {
  connection_id          = huaweicloud_dli_datasource_connection.test.id
  elastic_resource_pools = [huaweicloud_dli_elastic_resource_pool.test.name]

  depends_on = [huaweicloud_dli_queue.test]
}
```

**参数说明**：

- **name**：通过引用输入变量 `datasource_connection_name` 进行赋值，用于指定增强型数据源连接的名称
- **vpc_id**：通过引用 `huaweicloud_vpc.test.id` 进行赋值，用于指定数据源连接所属的虚拟私有云
- **subnet_id**：通过引用 `huaweicloud_vpc_subnet.test.id` 进行赋值，用于指定数据源连接所属的子网
- **connection_id**：通过引用 `huaweicloud_dli_datasource_connection.test.id` 进行赋值，用于指定待关联的数据源连接
- **elastic_resource_pools**：通过引用 `huaweicloud_dli_elastic_resource_pool.test.name` 进行赋值，用于指定关联的弹性资源池
- **depends_on**：显式声明依赖关系，确保队列创建完成后再执行关联操作

### 13. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name                   = "tf_test_mysql_dws_vpc"
vpc_cidr                   = "192.168.0.0/16"
subnet_name                = "tf_test_mysql_dws_subnet"
security_group_name        = "tf_test_mysql_dws_sg"
elastic_resource_pool_cidr = "172.16.0.0/18"
rds_instance_name          = "tf-test-rds-mysql"
rds_db_password            = "YourPassword@123"
dws_cluster_name           = "tf-test-dws-cluster"
dws_admin_user_pwd         = "YourPassword@123"
elastic_resource_pool_name = "tf_test_dli_pool"
queue_name                 = "tf_test_dli_queue"
datasource_connection_name = "tf_test_dli_conn"
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

### 14. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建MySQL到DWS实时数据同步所需的基础设施
4. 运行 `terraform show` 查看已创建的资源

> 注意：基础设施创建完成后，还需在 RDS MySQL 中创建源表、在 DWS 中创建目标表，准备 DWS Connector JAR 并上传至 OBS，然后在 DLI 中创建 Flink OpenSource SQL 作业，使用 `mysql-cdc` 作为源连接器、`gaussdb` 作为目标连接器，实现 MySQL 到 DWS 的实时数据同步。

## 参考信息

- [华为云数据仓库服务产品文档](https://support.huaweicloud.com/dws/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DWS MySQL到DWS实时同步最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dws/real-time-sync-mysql-to-dws)
