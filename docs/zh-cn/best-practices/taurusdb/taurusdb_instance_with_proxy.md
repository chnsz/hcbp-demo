# 部署TaurusDB实例并开启数据库代理

## 应用场景

云数据库TaurusDB是华为云提供的企业级云原生数据库服务，完全兼容MySQL协议，采用计算与存储分离架构，支持一写多读与秒级弹性伸缩。数据库代理（Database Proxy）位于应用与数据库实例之间，提供读写分离、连接池、事务拆分与访问控制等能力，可显著提升高并发场景下的连接复用率与读请求吞吐能力。

本最佳实践将介绍如何使用Terraform自动化部署TaurusDB实例并开启数据库代理，包括VPC与子网创建、安全组配置、实例规格与可用区查询、TaurusDB实例创建、代理规格查询以及数据库代理创建与节点权重、访问控制、代理参数配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [TaurusDB实例规格（data.huaweicloud_taurusdb_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_flavors)
- [TaurusDB代理规格（data.huaweicloud_taurusdb_proxy_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_proxy_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [TaurusDB实例（huaweicloud_taurusdb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_instance)
- [TaurusDB数据库代理（huaweicloud_taurusdb_proxy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_proxy)

### 资源/数据源依赖关系

```
data.huaweicloud_taurusdb_flavors
    ├── huaweicloud_taurusdb_instance
    │       └── data.huaweicloud_taurusdb_proxy_flavors
    │               └── huaweicloud_taurusdb_proxy
    └── (local.available_azs)

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            ├── huaweicloud_taurusdb_instance
            └── huaweicloud_taurusdb_proxy

huaweicloud_networking_secgroup
    └── huaweicloud_taurusdb_instance

random_password
    └── huaweicloud_taurusdb_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云和子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云和子网
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC网段，通过引用输入变量vpc_cidr进行赋值，未指定时默认为192.168.0.0/16
- **vpc_id**：子网所属的VPC ID，引用上一步创建的VPC的ID
- **cidr**（子网）：子网网段，通过引用输入变量subnet_cidr进行赋值，为空时基于VPC网段自动划分子网
- **gateway_ip**：子网网关IP，通过引用输入变量gateway_ip进行赋值，为空时自动取子网第一个可用IP

### 3. 查询TaurusDB实例规格与可用区

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询TaurusDB实例规格
variable "master_availability_zone" {
  description = "The master availability zone of the TaurusDB instance. If not specified, the first available AZ from flavors will be used"
  type        = string
  default     = ""
}

data "huaweicloud_taurusdb_flavors" "test" {
  engine                 = "gaussdb-mysql"
  version                = "8.0"
  availability_zone_mode = "multi"
}

locals {
  # 从规格的az_status中获取可用区（TaurusDB可用区必须来自flavors，而非huaweicloud_availability_zones）
  available_azs = try([for k, v in data.huaweicloud_taurusdb_flavors.test.flavors[0].az_status : k if v == "normal"], [])
  master_az     = var.master_availability_zone != "" ? var.master_availability_zone : try(local.available_azs[0], "")
}
```

**参数说明**：
- **engine**：数据库引擎，固定为gaussdb-mysql
- **version**：数据库版本，固定为8.0
- **availability_zone_mode**：可用区模式，设置为multi以查询多可用区规格
- **master_az**：主节点可用区，优先使用输入变量master_availability_zone，未指定时取规格中第一个可用区

### 4. 创建安全组

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true以便按需自定义访问规则

### 5. 生成实例随机密码

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下生成实例随机密码
variable "instance_password" {
  description = "The password for the TaurusDB instance"
  type        = string
  default     = ""
  sensitive   = true
}

resource "random_password" "test" {
  count = var.instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "!@%^*-_=+"
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**参数说明**：
- **count**：当输入变量instance_password为空时创建随机密码，否则使用用户指定的密码
- **length**：密码长度，设置为12
- **override_special**：允许使用的特殊字符集合
- **min_upper/min_lower/min_numeric/min_special**：密码中大写字母、小写字母、数字与特殊字符的最小数量

### 6. 创建TaurusDB实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建TaurusDB实例
variable "instance_name" {
  description = "The TaurusDB instance name"
  type        = string
}

variable "instance_flavor_ref" {
  description = "The flavor code of the TaurusDB instance. If not specified, the first available flavor will be used"
  type        = string
  default     = ""
}

variable "read_replicas" {
  description = "The number of read replicas"
  type        = number
  default     = 4
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = "0"
}

variable "volume_type" {
  description = "The storage type of the instance. Valid values are DL6, DL5"
  type        = string
  default     = "DL6"
}

variable "time_zone" {
  description = "The time zone of the instance"
  type        = string
  default     = "UTC+08:00"
}

variable "instance_db_port" {
  description = "The database port"
  type        = number
  default     = 3306
}

variable "ssl_option" {
  description = "Whether to enable SSL. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "sql_filter_enabled" {
  description = "Whether to enable SQL filter"
  type        = bool
  default     = true
}

variable "slow_log_show_original_switch" {
  description = "Whether to enable slow log show original switch"
  type        = bool
  default     = true
}

variable "table_name_case_sensitivity" {
  description = "Whether the kernel table name is case sensitive"
  type        = bool
  default     = true
}

variable "multi_tenant_switch" {
  description = "Whether to enable multi-tenancy switch. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "maintain_begin" {
  description = "The start time of the maintenance window in HH:MM format"
  type        = string
  default     = "02:00"
}

variable "maintain_end" {
  description = "The end time of the maintenance window in HH:MM format"
  type        = string
  default     = "06:00"
}

variable "description" {
  description = "The description of the TaurusDB instance"
  type        = string
  default     = ""
}

variable "seconds_level_monitoring_enabled" {
  description = "Whether to enable seconds level monitoring"
  type        = bool
  default     = true
}

variable "seconds_level_monitoring_period" {
  description = "The seconds level collection period. Valid values are 1, 5"
  type        = number
  default     = 5
}

variable "audit_log_enabled" {
  description = "Whether to enable audit log"
  type        = bool
  default     = true
}

variable "audit_log_keep_days" {
  description = "The number of days for storing audit logs"
  type        = number
  default     = 7
}

variable "reserve_audit_logs" {
  description = "Whether to reserve historical audit logs when SQL audit is disabled. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "instance_backup_time_window" {
  description = "The backup time window in HH:MM-HH:MM format"
  type        = string
}

variable "instance_backup_keep_days" {
  description = "The number of days to retain backups"
  type        = number
}

variable "tags" {
  description = "The tags of the TaurusDB instance"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_taurusdb_instance" "test" {
  name                             = var.instance_name
  password                         = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  flavor                           = var.instance_flavor_ref != "" ? var.instance_flavor_ref : try(data.huaweicloud_taurusdb_flavors.test.flavors[0].name, "")
  vpc_id                           = huaweicloud_vpc.test.id
  subnet_id                        = huaweicloud_vpc_subnet.test.id
  security_group_id                = huaweicloud_networking_secgroup.test.id
  mode                             = "Cluster"
  availability_zone_mode           = "multi"
  master_availability_zone         = local.master_az
  read_replicas                    = var.read_replicas
  enterprise_project_id            = var.enterprise_project_id
  volume_type                      = var.volume_type
  time_zone                        = var.time_zone
  port                             = var.instance_db_port
  ssl_option                       = var.ssl_option
  sql_filter_enabled               = var.sql_filter_enabled
  slow_log_show_original_switch    = var.slow_log_show_original_switch
  table_name_case_sensitivity      = var.table_name_case_sensitivity
  multi_tenant_switch              = var.multi_tenant_switch
  maintain_begin                   = var.maintain_begin
  maintain_end                     = var.maintain_end
  description                      = var.description
  seconds_level_monitoring_enabled = var.seconds_level_monitoring_enabled
  seconds_level_monitoring_period  = var.seconds_level_monitoring_enabled ? var.seconds_level_monitoring_period : null
  audit_log_enabled                = var.audit_log_enabled
  audit_log_keep_days              = var.audit_log_keep_days
  reserve_audit_logs               = var.reserve_audit_logs

  datastore {
    engine  = "gaussdb-mysql"
    version = "8.0"
  }

  backup_strategy {
    start_time = var.instance_backup_time_window
    keep_days  = tostring(var.instance_backup_keep_days)
  }

  tags = var.tags

  lifecycle {
    ignore_changes = [
      password, ssl_option, reserve_audit_logs, datastore[0].version,
    ]
  }
}
```

**参数说明**：
- **name**：实例名称，通过引用输入变量instance_name进行赋值
- **password**：实例密码，优先使用输入变量instance_password，为空时使用随机密码
- **flavor**：实例规格，优先使用输入变量instance_flavor_ref，未指定时取规格查询结果中的第一个规格
- **vpc_id/subnet_id/security_group_id**：分别引用前面创建的VPC、子网与安全组的ID
- **mode**：实例模式，设置为Cluster以支持数据库代理
- **availability_zone_mode**：可用区模式，设置为multi以支持多可用区部署
- **master_availability_zone**：主节点可用区，引用local.master_az
- **read_replicas**：只读节点数量，通过引用输入变量read_replicas进行赋值，使用数据库代理时建议至少4个
- **port**：数据库端口，通过引用输入变量instance_db_port进行赋值
- **datastore**：数据库引擎与版本，固定为gaussdb-mysql 8.0
- **backup_strategy**：备份策略，start_time与keep_days分别引用输入变量instance_backup_time_window与instance_backup_keep_days
- **tags**：实例标签，通过引用输入变量tags进行赋值

### 7. 查询数据库代理规格

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询数据库代理规格
data "huaweicloud_taurusdb_proxy_flavors" "test" {
  instance_id = huaweicloud_taurusdb_instance.test.id
}

locals {
  # 按节点名称排序，以便为节点分配一致的权重
  sort_nodes = tolist(values({ for node in huaweicloud_taurusdb_instance.test.nodes : node.name => node }))
}
```

**参数说明**：
- **instance_id**：TaurusDB实例ID，引用上一步创建的实例的ID
- **sort_nodes**：对实例节点按名称排序，用于后续为代理主节点与只读节点分配权重

### 8. 创建数据库代理

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建数据库代理
variable "proxy_node_num" {
  description = "The number of proxy nodes"
  type        = number
  default     = 2
}

variable "proxy_name" {
  description = "The name of the database proxy"
  type        = string
  default     = ""
}

variable "proxy_mode" {
  description = "The type of the proxy. Valid values are readwrite, readonly. Defaults to readwrite"
  type        = string
  default     = "readwrite"
}

variable "route_mode" {
  description = "The routing policy of the proxy. Valid values are 0, 1, 2"
  type        = number
  default     = 1
}

variable "proxy_new_node_auto_add_status" {
  description = "Whether to automatically add new nodes to the proxy. Valid values are ON, OFF"
  type        = string
  default     = "OFF"
}

variable "proxy_new_node_weight" {
  description = "The weight of new nodes automatically added to the proxy"
  type        = number
  default     = 20
}

variable "proxy_port" {
  description = "The proxy port"
  type        = number
  default     = 3306
}

variable "proxy_transaction_split" {
  description = "Whether to enable transaction split. Valid values are ON, OFF. Defaults to OFF"
  type        = string
  default     = "OFF"
}

variable "proxy_consistence_mode" {
  description = "The consistency mode. Valid values are session, global, eventual. Defaults to eventual"
  type        = string
  default     = "eventual"
}

variable "proxy_connection_pool_type" {
  description = "The connection pool type. Valid values are SESSION, CLOSED. Defaults to CLOSED"
  type        = string
  default     = "CLOSED"
}

variable "proxy_open_access_control" {
  description = "Whether to enable access control"
  type        = bool
  default     = true
}

variable "access_control_type" {
  description = "The access control mode. Valid values are white, black"
  type        = string
  default     = "white"
}

variable "proxy_dns_name_prefix" {
  description = "The DNS name prefix for the proxy"
  type        = string
  default     = ""
}

variable "proxy_access_control_ip_list" {
  description = "The access control IP list for the proxy"
  type        = list(object({
    ip          = string
    description = string
  }))
  default     = []
}

variable "proxy_master_node_weight" {
  description = "The weight of the master node in the proxy"
  type        = number
  default     = 20
}

variable "proxy_readonly_node_weight" {
  description = "The weight of read-only nodes in the proxy"
  type        = number
  default     = 30
}

variable "proxy_parameters" {
  description = "The parameters for the proxy"
  type        = list(object({
    name      = string
    value     = string
    elem_type = string
  }))
  default     = []
}

resource "huaweicloud_taurusdb_proxy" "test" {
  instance_id              = huaweicloud_taurusdb_instance.test.id
  flavor                   = try(data.huaweicloud_taurusdb_proxy_flavors.test.flavor_groups[0].flavors[0].spec_code, "")
  node_num                 = var.proxy_node_num
  proxy_name               = var.proxy_name
  proxy_mode               = var.proxy_mode
  route_mode               = var.route_mode
  subnet_id                = huaweicloud_vpc_subnet.test.id
  new_node_auto_add_status = var.proxy_new_node_auto_add_status
  new_node_weight          = var.proxy_new_node_weight
  port                     = var.proxy_port
  transaction_split        = var.proxy_transaction_split
  consistence_mode         = var.proxy_consistence_mode
  connection_pool_type     = var.proxy_connection_pool_type
  open_access_control      = var.proxy_open_access_control
  access_control_type      = var.proxy_open_access_control ? var.access_control_type : null
  dns_name_prefix          = var.proxy_dns_name_prefix

  dynamic "access_control_ip_list" {
    for_each = var.proxy_access_control_ip_list

    content {
      ip          = access_control_ip_list.value.ip
      description = access_control_ip_list.value.description
    }
  }

  master_node_weight {
    id     = try(local.sort_nodes[0].id, "")
    weight = var.proxy_master_node_weight
  }

  dynamic "readonly_nodes_weight" {
    for_each = length(local.sort_nodes) > 1 ? slice(local.sort_nodes, 1, length(local.sort_nodes)) : []

    content {
      id     = readonly_nodes_weight.value.id
      weight = var.proxy_readonly_node_weight
    }
  }

  dynamic "parameters" {
    for_each = var.proxy_parameters

    content {
      name      = parameters.value.name
      value     = parameters.value.value
      elem_type = parameters.value.elem_type
    }
  }

  lifecycle {
    ignore_changes = [
      new_node_weight, proxy_mode, readonly_nodes_weight, parameters,
    ]
  }
}
```

**参数说明**：
- **instance_id**：代理所属的TaurusDB实例ID，引用上一步创建的实例的ID
- **flavor**：代理规格，引用代理规格查询结果中的第一个规格
- **node_num**：代理节点数量，通过引用输入变量proxy_node_num进行赋值
- **proxy_name**：代理名称，通过引用输入变量proxy_name进行赋值
- **proxy_mode**：代理模式，通过引用输入变量proxy_mode进行赋值，支持readwrite与readonly
- **route_mode**：路由策略，通过引用输入变量route_mode进行赋值
- **subnet_id**：代理所属子网ID，引用前面创建的子网的ID
- **new_node_auto_add_status/new_node_weight**：新节点自动加入状态与新节点权重，分别引用输入变量proxy_new_node_auto_add_status与proxy_new_node_weight
- **port**：代理端口，通过引用输入变量proxy_port进行赋值
- **transaction_split**：是否开启事务拆分，通过引用输入变量proxy_transaction_split进行赋值
- **consistence_mode**：一致性模式，通过引用输入变量proxy_consistence_mode进行赋值
- **connection_pool_type**：连接池类型，通过引用输入变量proxy_connection_pool_type进行赋值
- **open_access_control/access_control_type**：是否开启访问控制与访问控制模式，分别引用输入变量proxy_open_access_control与access_control_type
- **dns_name_prefix**：代理DNS名称前缀，通过引用输入变量proxy_dns_name_prefix进行赋值
- **access_control_ip_list**：访问控制IP列表，通过引用输入变量proxy_access_control_ip_list进行赋值
- **master_node_weight**：主节点权重，id引用排序后的第一个节点，weight引用输入变量proxy_master_node_weight
- **readonly_nodes_weight**：只读节点权重，id引用排序后的其余节点，weight引用输入变量proxy_readonly_node_weight
- **parameters**：代理参数，通过引用输入变量proxy_parameters进行赋值

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name                     = "your_vpc"
subnet_name                  = "your_subnet"
security_group_name          = "your_security_group"
instance_name                = "your_taurusdb_instance"
instance_backup_time_window  = "09:00-10:00"
instance_backup_keep_days    = 7
proxy_name                   = "your_proxy"
proxy_node_num               = 2
proxy_port                   = 3339
proxy_access_control_ip_list = [
  {
    ip          = "3.3.3.3"
    description = "test description"
  }
]
proxy_parameters             = [
  {
    name      = "multiStatementType"
    value     = "Loose"
    elem_type = "system"
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

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建TaurusDB实例与数据库代理
4. 运行 `terraform show` 查看已创建的TaurusDB实例与数据库代理

## 参考信息

- [华为云TaurusDB产品文档](https://support.huaweicloud.com/taurusdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [TaurusDB实例与数据库代理最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/taurusdb/taurusdb-instance-with-proxy)
