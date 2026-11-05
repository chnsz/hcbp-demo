# 部署TaurusDB实例并关联LTS日志

## 应用场景

云数据库TaurusDB是华为云提供的企业级云原生数据库服务，完全兼容MySQL协议，采用计算与存储分离架构，支持一写多读、秒级弹性伸缩与海量数据存储。在实际生产环境中，数据库的错误日志与慢日志是定位故障、分析性能瓶颈的重要依据，若仅保存在实例本地，既不便于长期留存，也不便于集中检索与分析。

本最佳实践将介绍如何使用Terraform自动化部署TaurusDB实例，并创建云日志服务（LTS）日志组与日志流，通过TaurusDB的LTS日志关联能力将实例日志集中采集至LTS，实现日志的统一存储、检索与生命周期管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [TaurusDB实例规格（data.huaweicloud_taurusdb_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [TaurusDB实例（huaweicloud_taurusdb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_instance)
- [LTS日志组（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS日志流（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [TaurusDB LTS日志（huaweicloud_taurusdb_lts_log）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_lts_log)

### 资源/数据源依赖关系

```
data.huaweicloud_taurusdb_flavors.test
    ├── huaweicloud_taurusdb_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
            └── huaweicloud_taurusdb_instance.test

huaweicloud_networking_secgroup.test
    ├── huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_taurusdb_instance.test

random_password.test
    └── huaweicloud_taurusdb_instance.test

huaweicloud_lts_group.test
    ├── huaweicloud_lts_stream.test
    └── huaweicloud_taurusdb_lts_log.test

huaweicloud_lts_stream.test
    └── huaweicloud_taurusdb_lts_log.test

huaweicloud_taurusdb_instance.test
    └── huaweicloud_taurusdb_lts_log.test
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
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，通过引用输入变量 vpc_cidr 进行赋值
- **vpc_id**：子网所属的VPC ID，引用上一步创建的VPC的ID进行赋值
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网网段，当输入变量 subnet_cidr 为空时，使用 `cidrsubnet` 函数从VPC网段中自动划分
- **gateway_ip**：子网网关IP，当输入变量 gateway_ip 为空时，使用 `cidrhost` 函数自动计算

### 3. 查询TaurusDB实例规格

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询TaurusDB实例规格
variable "availability_zone_mode" {
  description = "The availability zone mode. Valid values are single, multi"
  type        = string
  default     = "multi"
}

variable "master_availability_zone" {
  description = "The master availability zone of the TaurusDB instance. If not specified, the first available AZ from flavors will be used"
  type        = string
  default     = ""
}

data "huaweicloud_taurusdb_flavors" "test" {
  engine                 = "gaussdb-mysql"
  version                = "8.0"
  availability_zone_mode = var.availability_zone_mode
}

locals {
  # 从规格的az_status中获取可用区（TaurusDB的可用区必须来自规格查询结果）
  available_azs = try([for k, v in data.huaweicloud_taurusdb_flavors.test.flavors[0].az_status : k if v == "normal"], [])
  master_az     = var.master_availability_zone != "" ? var.master_availability_zone : try(local.available_azs[0], "")
}
```

**参数说明**：
- **engine**：数据库引擎，固定为 `gaussdb-mysql`
- **version**：数据库引擎版本，固定为 `8.0`
- **availability_zone_mode**：可用区模式，通过引用输入变量 availability_zone_mode 进行赋值
- **available_azs**：本地变量，从规格查询结果的 `az_status` 中筛选出状态为 `normal` 的可用区
- **master_az**：本地变量，当输入变量 master_availability_zone 为空时，取第一个可用区作为主可用区

### 4. 创建安全组和访问规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组及其访问规则
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

variable "instance_db_port" {
  description = "The database port"
  type        = number
  default     = 3306
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = var.vpc_cidr
  ports             = var.instance_db_port
  protocol          = "tcp"
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：是否删除安全组默认规则，设置为 `true` 以便仅保留显式定义的规则
- **security_group_id**：规则所属的安全组ID，引用上一步创建的安全组的ID进行赋值
- **direction**：规则方向，设置为 `ingress` 表示入方向
- **remote_ip_prefix**：允许访问的源网段，通过引用输入变量 vpc_cidr 进行赋值
- **ports**：允许访问的端口，通过引用输入变量 instance_db_port 进行赋值
- **protocol**：协议类型，设置为 `tcp`

### 5. 创建随机密码

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建随机密码
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
- **count**：当输入变量 instance_password 为空时创建随机密码，否则不创建
- **length**：密码长度，设置为 `12`
- **special**：是否包含特殊字符，设置为 `true`
- **override_special**：允许使用的特殊字符集合
- **min_upper**、**min_lower**、**min_numeric**、**min_special**：分别表示大写字母、小写字母、数字和特殊字符的最小数量

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

variable "instance_mode" {
  description = "The instance mode. Valid values are Cluster, StandSingle"
  type        = string
  default     = "Cluster"
}

variable "read_replicas" {
  description = "The number of read replicas"
  type        = number
  default     = 2
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
  mode                             = var.instance_mode
  availability_zone_mode           = var.availability_zone_mode
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
- **name**：实例名称，通过引用输入变量 instance_name 进行赋值
- **password**：实例密码，当输入变量 instance_password 不为空时使用该值，否则使用随机密码资源生成的结果
- **flavor**：实例规格，当输入变量 instance_flavor_ref 不为空时使用该值，否则使用规格查询结果中的第一个规格
- **vpc_id**、**subnet_id**、**security_group_id**：分别引用前面创建的VPC、子网和安全组的ID进行赋值
- **mode**：实例模式，通过引用输入变量 instance_mode 进行赋值
- **availability_zone_mode**、**master_availability_zone**：可用区模式与主可用区，分别引用输入变量 availability_zone_mode 和本地变量 master_az 进行赋值
- **read_replicas**：只读节点数量，通过引用输入变量 read_replicas 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **volume_type**：存储类型，通过引用输入变量 volume_type 进行赋值
- **time_zone**：时区，通过引用输入变量 time_zone 进行赋值
- **port**：数据库端口，通过引用输入变量 instance_db_port 进行赋值
- **ssl_option**：是否开启SSL，通过引用输入变量 ssl_option 进行赋值
- **sql_filter_enabled**、**slow_log_show_original_switch**、**table_name_case_sensitivity**、**multi_tenant_switch**：分别通过引用对应的输入变量进行赋值
- **maintain_begin**、**maintain_end**：可维护时间窗的起止时间，分别通过引用输入变量 maintain_begin 和 maintain_end 进行赋值
- **description**：实例描述，通过引用输入变量 description 进行赋值
- **seconds_level_monitoring_enabled**、**seconds_level_monitoring_period**：秒级监控开关与采集周期，分别通过引用对应的输入变量进行赋值
- **audit_log_enabled**、**audit_log_keep_days**、**reserve_audit_logs**：审计日志相关配置，分别通过引用对应的输入变量进行赋值
- **datastore**：数据库引擎信息，引擎固定为 `gaussdb-mysql`，版本固定为 `8.0`
- **backup_strategy**：备份策略，`start_time` 引用输入变量 instance_backup_time_window，`keep_days` 引用输入变量 instance_backup_keep_days
- **tags**：实例标签，通过引用输入变量 tags 进行赋值

### 7. 创建LTS日志组和日志流

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建LTS日志组和日志流
variable "lts_group_name" {
  description = "The name of the LTS log group"
  type        = string
}

variable "lts_group_ttl_in_days" {
  description = "The expiration time (in days) of the LTS log group. The value is range from 1 to 365"
  type        = number
  default     = 30
}

variable "lts_stream_name" {
  description = "The name of the LTS log stream"
  type        = string
}

variable "lts_stream_ttl_in_days" {
  description = "The expiration time (in days) of the LTS log stream. The valid value is from 1 to 365"
  type        = number
  default     = 7
}

variable "lts_stream_is_favorite" {
  description = "Whether to favorite the log stream"
  type        = bool
  default     = false
}

resource "huaweicloud_lts_group" "test" {
  group_name  = var.lts_group_name
  ttl_in_days = var.lts_group_ttl_in_days
}

resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.lts_stream_name
  ttl_in_days = var.lts_stream_ttl_in_days
  is_favorite = var.lts_stream_is_favorite
}
```

**参数说明**：
- **group_name**：日志组名称，通过引用输入变量 lts_group_name 进行赋值
- **ttl_in_days**：日志组日志保存天数，通过引用输入变量 lts_group_ttl_in_days 进行赋值
- **group_id**：日志流所属的日志组ID，引用上一步创建的日志组的ID进行赋值
- **stream_name**：日志流名称，通过引用输入变量 lts_stream_name 进行赋值
- **ttl_in_days**：日志流日志保存天数，通过引用输入变量 lts_stream_ttl_in_days 进行赋值
- **is_favorite**：是否收藏日志流，通过引用输入变量 lts_stream_is_favorite 进行赋值

### 8. 关联TaurusDB实例与LTS日志流

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下关联TaurusDB实例与LTS日志流
variable "lts_log_type" {
  description = "The type of the TaurusDB LTS log. Valid values are error_log, slow_log"
  type        = string
  default     = "error_log"
}

resource "huaweicloud_taurusdb_lts_log" "test" {
  instance_id   = huaweicloud_taurusdb_instance.test.id
  log_type      = var.lts_log_type
  lts_group_id  = huaweicloud_lts_group.test.id
  lts_stream_id = huaweicloud_lts_stream.test.id
}
```

**参数说明**：
- **instance_id**：TaurusDB实例ID，引用前面创建的TaurusDB实例的ID进行赋值
- **log_type**：日志类型，通过引用输入变量 lts_log_type 进行赋值，可选值为 `error_log` 和 `slow_log`
- **lts_group_id**：LTS日志组ID，引用前面创建的日志组的ID进行赋值
- **lts_stream_id**：LTS日志流ID，引用前面创建的日志流的ID进行赋值

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证参数
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源参数
vpc_name                    = "your_vpc"
subnet_name                 = "your_subnet"
security_group_name         = "your_security_group"
instance_name               = "your_taurusdb_instance"
instance_backup_time_window = "02:00-03:00"
instance_backup_keep_days   = 7
lts_group_name              = "your_lts_group"
lts_stream_name             = "your_lts_stream"
tags = {
  foo = "bar"
  key = "value"
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

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建TaurusDB实例及其关联的LTS日志
4. 运行 `terraform show` 查看已创建的TaurusDB实例及其关联的LTS日志

## 参考信息

- [华为云TaurusDB产品文档](https://support.huaweicloud.com/taurusdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [TaurusDB实例关联LTS日志最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/taurusdb/taurusdb-instance-with-lts-log)
