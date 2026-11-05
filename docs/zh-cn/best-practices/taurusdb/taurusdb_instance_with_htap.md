# 部署TaurusDB实例并挂载HTAP StarRocks实例

## 应用场景

云数据库TaurusDB是华为云提供的企业级云原生数据库服务，完全兼容MySQL协议，采用计算与存储分离架构，适用于在线事务处理（OLTP）等对性能、可靠性和扩展性要求较高的业务场景。当业务需要在同一数据库实例上同时获得高性能事务处理与实时分析能力时，可以为TaurusDB实例挂载HTAP StarRocks实例，实现事务与分析一体化（HTAP）。

本最佳实践将介绍如何使用Terraform自动化部署TaurusDB实例并挂载HTAP StarRocks实例，包括VPC与子网创建、安全组配置、TaurusDB实例规格与可用区查询、TaurusDB实例创建、HTAP StarRocks实例规格与引擎版本查询，以及HTAP StarRocks实例的创建与参数配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [TaurusDB实例规格（data.huaweicloud_taurusdb_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_flavors)
- [TaurusDB HTAP实例规格（data.huaweicloud_taurusdb_htap_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_htap_flavors)
- [TaurusDB HTAP引擎版本（data.huaweicloud_taurusdb_htap_datastores）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_htap_datastores)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [TaurusDB实例（huaweicloud_taurusdb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_instance)
- [TaurusDB HTAP StarRocks实例（huaweicloud_taurusdb_htap_starrocks_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_htap_starrocks_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_taurusdb_instance

data.huaweicloud_taurusdb_flavors
    └── huaweicloud_taurusdb_instance

data.huaweicloud_taurusdb_htap_flavors
    └── huaweicloud_taurusdb_htap_starrocks_instance

data.huaweicloud_taurusdb_htap_datastores
    └── huaweicloud_taurusdb_htap_starrocks_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_taurusdb_instance
            └── huaweicloud_taurusdb_htap_starrocks_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_taurusdb_instance
    └── huaweicloud_taurusdb_htap_starrocks_instance

random_password
    ├── huaweicloud_taurusdb_instance
    └── huaweicloud_taurusdb_htap_starrocks_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云和子网

在TF文件（如main.tf）中添加以下脚本以创建TaurusDB实例和HTAP StarRocks实例所需的网络：

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
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量 vpc_cidr 进行赋值，未指定时默认为 `192.168.0.0/16`
- **vpc_id**：子网所属的VPC ID，引用上一步创建的VPC的ID进行赋值
- **cidr**（子网）：子网网段，未通过输入变量 subnet_cidr 指定时，基于VPC网段自动划分
- **gateway_ip**：子网网关IP，未通过输入变量 gateway_ip 指定时，自动取子网网段的第一个可用IP

### 3. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建TaurusDB实例和HTAP StarRocks实例使用的安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The security group name of the TaurusDB instance"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：是否删除安全组默认规则，设置为 `true` 以便按需自定义访问规则

### 4. 生成随机密码

在TF文件（如main.tf）中添加以下脚本以生成TaurusDB实例和HTAP StarRocks实例的初始密码：

```hcl
# 生成随机密码，用于TaurusDB实例和HTAP StarRocks实例
resource "random_password" "test" {
  length           = 16
  special          = true
  override_special = "!@%^*-_=+"
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**参数说明**：
- **length**：密码长度，设置为 `16`
- **special**：是否包含特殊字符，设置为 `true`
- **override_special**：允许使用的特殊字符集合
- **min_upper** / **min_lower** / **min_numeric** / **min_special**：分别要求至少包含的大写字母、小写字母、数字和特殊字符数量

### 5. 查询可用区、TaurusDB实例规格与HTAP引擎资源

在TF文件（如main.tf）中添加以下脚本以查询部署所需的可用区、实例规格和引擎版本信息：

```hcl
# 查询可用区、TaurusDB实例规格与HTAP引擎资源
variable "taurusdb_availability_zone_mode" {
  description = "The availability zone mode of the TaurusDB instance. Valid values are single, multi"
  type        = string
  default     = "multi"
}

variable "az_code" {
  description = "The AZ code of the HTAP StarRocks instance. If not specified, the first available AZ from flavors will be used"
  type        = string
  default     = ""
}

variable "fe_flavor_id" {
  description = "The specification ID of the frontend node. If not specified, the first available flavor will be used"
  type        = string
  default     = ""
}

variable "be_flavor_id" {
  description = "The specification ID of the backend node. If not specified, the first available flavor will be used"
  type        = string
  default     = ""
}

variable "engine_version" {
  description = "The major version number of the engine. If not specified, the latest version from the data source will be used"
  type        = string
  default     = ""
}

data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_taurusdb_flavors" "test" {
  engine                 = "gaussdb-mysql"
  version                = "8.0"
  availability_zone_mode = var.taurusdb_availability_zone_mode
}

data "huaweicloud_taurusdb_htap_flavors" "test" {
  engine_name            = "star-rocks"
  availability_zone_mode = "single"
}

data "huaweicloud_taurusdb_htap_datastores" "test" {
  engine_name = "star-rocks"
}

locals {
  # 从HTAP实例规格的az_status中获取第一个可用区
  htap_az_status = try(data.huaweicloud_taurusdb_htap_flavors.test.flavors[0].az_status, {})
  all_az_normal  = [for k, v in local.htap_az_status : k if v == "normal"]
  az_code        = var.az_code != "" ? var.az_code : try(local.all_az_normal[0], "")

  # 按规格编码和可用区状态过滤前端和后端规格
  be_flavors = [for f in data.huaweicloud_taurusdb_htap_flavors.test.flavors : f if length(regexall("sr-be", f.spec_code)) > 0 && contains(keys(f.az_status), local.az_code) && f.az_status[local.az_code] == "normal"]
  fe_flavors = [for f in data.huaweicloud_taurusdb_htap_flavors.test.flavors : f if length(regexall("sr-fe", f.spec_code)) > 0 && contains(keys(f.az_status), local.az_code) && f.az_status[local.az_code] == "normal"]

  fe_flavor_id   = var.fe_flavor_id != "" ? var.fe_flavor_id : try(local.fe_flavors[0].id, "")
  be_flavor_id   = var.be_flavor_id != "" ? var.be_flavor_id : try(local.be_flavors[0].id, "")
  engine_version = var.engine_version != "" ? var.engine_version : try(data.huaweicloud_taurusdb_htap_datastores.test.datastores[0].name, "")
}
```

**参数说明**：
- **engine**：TaurusDB实例引擎，设置为 `gaussdb-mysql`
- **version**：TaurusDB实例引擎版本，设置为 `8.0`
- **availability_zone_mode**：TaurusDB实例可用区模式，通过引用输入变量 taurusdb_availability_zone_mode 进行赋值
- **engine_name**：HTAP实例引擎名称，设置为 `star-rocks`
- **locals.az_code**：HTAP StarRocks实例的可用区编码，未通过输入变量 az_code 指定时，自动选择第一个状态为 `normal` 的可用区
- **locals.fe_flavor_id** / **locals.be_flavor_id**：HTAP StarRocks实例前端/后端节点规格ID，未通过输入变量指定时，自动从数据源结果中筛选可用规格
- **locals.engine_version**：HTAP StarRocks实例引擎版本，未通过输入变量 engine_version 指定时，自动取数据源返回的最新版本

### 6. 创建TaurusDB实例

在TF文件（如main.tf）中添加以下脚本以创建TaurusDB实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建TaurusDB实例
variable "taurusdb_instance_name" {
  description = "The TaurusDB instance name"
  type        = string
}

variable "taurusdb_flavor_ref" {
  description = "The flavor code of the TaurusDB instance. If not specified, the first available flavor will be used"
  type        = string
  default     = ""
}

variable "taurusdb_root_pwd" {
  description = "The database password of the TaurusDB instance. If not specified, a random password will be generated"
  type        = string
  default     = ""
  sensitive   = true
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = "0"
}

variable "taurusdb_read_replicas" {
  description = "The number of read replicas of the TaurusDB instance"
  type        = number
  default     = 2
}

resource "huaweicloud_taurusdb_instance" "test" {
  name                     = var.taurusdb_instance_name
  flavor                   = var.taurusdb_flavor_ref != "" ? var.taurusdb_flavor_ref : try(data.huaweicloud_taurusdb_flavors.test.flavors[0].name, "")
  vpc_id                   = huaweicloud_vpc.test.id
  subnet_id                = huaweicloud_vpc_subnet.test.id
  security_group_id        = huaweicloud_networking_secgroup.test.id
  password                 = var.taurusdb_root_pwd != "" ? var.taurusdb_root_pwd : random_password.test.result
  enterprise_project_id    = var.enterprise_project_id
  availability_zone_mode   = var.taurusdb_availability_zone_mode
  master_availability_zone = try(data.huaweicloud_availability_zones.test.names[0], "")
  read_replicas            = var.taurusdb_read_replicas

  datastore {
    engine  = "gaussdb-mysql"
    version = "8.0"
  }

  lifecycle {
    ignore_changes = [
      password, datastore[0].version,
    ]
  }
}
```

**参数说明**：
- **name**：TaurusDB实例名称，通过引用输入变量 taurusdb_instance_name 进行赋值
- **flavor**：TaurusDB实例规格，未通过输入变量 taurusdb_flavor_ref 指定时，自动取数据源返回的第一个可用规格
- **vpc_id** / **subnet_id**：实例所属的VPC和子网，分别引用前面创建的VPC和子网的ID进行赋值
- **security_group_id**：实例绑定的安全组，引用前面创建的安全组的ID进行赋值
- **password**：实例初始密码，未通过输入变量 taurusdb_root_pwd 指定时，使用随机生成的密码
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **availability_zone_mode**：实例可用区模式，通过引用输入变量 taurusdb_availability_zone_mode 进行赋值
- **master_availability_zone**：主节点可用区，引用可用区数据源返回的第一个可用区名称进行赋值
- **read_replicas**：只读节点数量，通过引用输入变量 taurusdb_read_replicas 进行赋值
- **datastore.engine** / **datastore.version**：实例引擎与版本，分别设置为 `gaussdb-mysql` 和 `8.0`

### 7. 创建HTAP StarRocks实例

在TF文件（如main.tf）中添加以下脚本以创建挂载到TaurusDB实例的HTAP StarRocks实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建HTAP StarRocks实例
variable "htap_instance_name" {
  description = "The HTAP StarRocks instance name. The name must start with a letter and consist of 4 to 64 characters"
  type        = string
}

variable "fe_count" {
  description = "The number of frontend nodes. For a single-node instance, the value is fixed to 1. For a cluster instance, the value ranges from 3 to 10"
  type        = number
  default     = 1
}

variable "be_count" {
  description = "The number of backend nodes. For a single-node instance, the value is fixed to 1. For a cluster instance, the value ranges from 3 to 10"
  type        = number
  default     = 1
}

variable "htap_db_root_pwd" {
  description = "The database password of the HTAP StarRocks instance. If not specified, a random password will be generated"
  type        = string
  default     = ""
  sensitive   = true
}

variable "time_zone" {
  description = "The time zone of the HTAP StarRocks instance"
  type        = string
  default     = "UTC+08:00"
}

variable "enable_users_sync" {
  description = "Whether to enable users synchronization. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "open_slow_log_switch" {
  description = "Whether to enable the slow query log original text switch. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "ha_mode" {
  description = "The deployment mode of the HTAP StarRocks instance. Valid values are Single, Cluster"
  type        = string
  default     = "Single"
}

variable "volume_io_type" {
  description = "The storage type of the frontend and backend nodes. Valid values are SSD, ESSD"
  type        = string
  default     = "SSD"
}

variable "fe_volume_capacity" {
  description = "The disk capacity in GB of the frontend node. The value ranges from 50 to 1000 and the increment is 10 GB"
  type        = number
  default     = 50
}

variable "be_volume_capacity" {
  description = "The disk capacity in GB of the backend node. The value ranges from 50 to 1000 and the increment is 10 GB"
  type        = number
  default     = 50
}

variable "be_parameter_values" {
  description = "A map contains mappings of parameter name and value to modify for the backend nodes"
  type        = map(string)
  default     = {}
}

variable "fe_parameter_values" {
  description = "A map contains mappings of parameter name and value to modify for the frontend nodes"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_taurusdb_htap_starrocks_instance" "test" {
  instance_id          = huaweicloud_taurusdb_instance.test.id
  name                 = var.htap_instance_name
  fe_flavor_id         = local.fe_flavor_id
  be_flavor_id         = local.be_flavor_id
  az_code              = local.az_code
  az_mode              = "single"
  fe_count             = var.fe_count
  be_count             = var.be_count
  db_root_pwd          = var.htap_db_root_pwd != "" ? var.htap_db_root_pwd : random_password.test.result
  time_zone            = var.time_zone
  security_group_id    = huaweicloud_networking_secgroup.test.id
  enable_users_sync    = var.enable_users_sync
  open_slow_log_switch = var.open_slow_log_switch

  engine {
    type    = "star-rocks"
    version = local.engine_version
  }

  ha {
    mode = var.ha_mode
  }

  fe_volume {
    io_type        = var.volume_io_type
    capacity_in_gb = var.fe_volume_capacity
  }

  be_volume {
    io_type        = var.volume_io_type
    capacity_in_gb = var.be_volume_capacity
  }

  tags_info {
    sys_tags {
      key   = "_sys_enterprise_project_id"
      value = var.enterprise_project_id
    }
  }

  be_parameter_values = var.be_parameter_values
  fe_parameter_values = var.fe_parameter_values

  lifecycle {
    ignore_changes = [
      db_root_pwd, be_parameter_values, fe_parameter_values,
    ]
  }
}
```

**参数说明**：
- **instance_id**：HTAP StarRocks实例挂载的TaurusDB实例ID，引用前面创建的TaurusDB实例的ID进行赋值
- **name**：HTAP StarRocks实例名称，通过引用输入变量 htap_instance_name 进行赋值
- **fe_flavor_id** / **be_flavor_id**：前端/后端节点规格ID，引用本地变量 local.fe_flavor_id 和 local.be_flavor_id 进行赋值
- **az_code**：实例可用区编码，引用本地变量 local.az_code 进行赋值
- **az_mode**：可用区模式，设置为 `single`
- **fe_count** / **be_count**：前端/后端节点数量，分别通过引用输入变量 fe_count 和 be_count 进行赋值
- **db_root_pwd**：实例初始密码，未通过输入变量 htap_db_root_pwd 指定时，使用随机生成的密码
- **time_zone**：实例时区，通过引用输入变量 time_zone 进行赋值
- **security_group_id**：实例绑定的安全组，引用前面创建的安全组的ID进行赋值
- **enable_users_sync**：是否开启用户同步，通过引用输入变量 enable_users_sync 进行赋值
- **open_slow_log_switch**：是否开启慢查询日志原文开关，通过引用输入变量 open_slow_log_switch 进行赋值
- **engine.type** / **engine.version**：实例引擎类型与版本，分别设置为 `star-rocks` 和本地变量 local.engine_version
- **ha.mode**：实例部署模式，通过引用输入变量 ha_mode 进行赋值
- **fe_volume** / **be_volume**：前端/后端节点存储配置，分别通过引用输入变量 volume_io_type、fe_volume_capacity 和 be_volume_capacity 进行赋值
- **tags_info.sys_tags**：实例系统标签，将企业项目ID写入 `_sys_enterprise_project_id` 标签
- **be_parameter_values** / **fe_parameter_values**：后端/前端节点参数配置，分别通过引用输入变量 be_parameter_values 和 fe_parameter_values 进行赋值

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
vpc_name               = "your_vpc"
subnet_name            = "your_subnet"
security_group_name    = "your_security_group"
taurusdb_instance_name = "your_taurusdb_instance"
htap_instance_name     = "your_htap_starrocks_instance"
be_parameter_values    = {
  alter_tablet_worker_count            = "1"
  base_compaction_num_threads_per_disk = "1"
}
fe_parameter_values    = {
  alter_table_timeout_second     = "21600"
  bdbje_heartbeat_timeout_second = "10"
}
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

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建TaurusDB实例及HTAP StarRocks实例
4. 运行 `terraform show` 查看已创建的TaurusDB实例及HTAP StarRocks实例

## 参考信息

- [华为云TaurusDB产品文档](https://support.huaweicloud.com/taurusdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [TaurusDB实例与HTAP StarRocks实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/taurusdb/taurusdb-instance-with-htap)
