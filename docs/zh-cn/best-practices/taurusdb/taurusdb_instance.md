# 部署TaurusDB实例

## 应用场景

TaurusDB是华为云提供的企业级云原生数据库服务，完全兼容MySQL协议，采用计算与存储分离架构，支持一写多读、秒级弹性扩展和并行查询等能力，适用于对性能、可靠性和扩展性要求较高的在线事务处理（OLTP）业务场景。

本最佳实践将介绍如何使用Terraform自动化部署一个TaurusDB实例，包括VPC、子网、安全组、参数模板、实例、账号和数据库的创建，帮助您快速构建一套可用的云原生数据库环境。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [TaurusDB规格（data.huaweicloud_taurusdb_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [TaurusDB参数模板（huaweicloud_taurusdb_parameter_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_parameter_template)
- [TaurusDB实例（huaweicloud_taurusdb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_instance)
- [TaurusDB账号（huaweicloud_taurusdb_account）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_account)
- [TaurusDB数据库（huaweicloud_taurusdb_database）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_database)

### 资源/数据源依赖关系

```
data.huaweicloud_taurusdb_flavors
    └── huaweicloud_taurusdb_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_taurusdb_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_taurusdb_instance

random_password
    ├── huaweicloud_taurusdb_instance
    └── huaweicloud_taurusdb_account

huaweicloud_taurusdb_parameter_template
    └── huaweicloud_taurusdb_instance

huaweicloud_taurusdb_instance
    ├── huaweicloud_taurusdb_account
    └── huaweicloud_taurusdb_database
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC

在TF文件（如main.tf）中添加以下脚本以创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，通过引用输入变量 vpc_cidr 进行赋值

### 3. 创建子网

在TF文件（如main.tf）中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建子网资源
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

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，引用前一步创建的VPC的ID进行赋值
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网的网段，当输入变量 subnet_cidr 为空时，基于VPC网段自动划分子网网段
- **gateway_ip**：子网的网关IP，当输入变量 gateway_ip 为空时，基于子网网段自动计算网关IP

### 4. 查询TaurusDB规格

在TF文件（如main.tf）中添加以下脚本以查询TaurusDB规格信息：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询TaurusDB规格数据源
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
  # Get the first available AZ from the flavor's az_status
  available_azs = try([for k, v in data.huaweicloud_taurusdb_flavors.test.flavors[0].az_status : k if v == "normal"], [])
  master_az     = var.master_availability_zone != "" ? var.master_availability_zone : try(local.available_azs[0], "")
}
```

**参数说明**：
- **engine**：数据库引擎，固定为 gaussdb-mysql
- **version**：数据库版本，固定为 8.0
- **availability_zone_mode**：可用区模式，通过引用输入变量 availability_zone_mode 进行赋值
- **locals.master_az**：主节点可用区，当输入变量 master_availability_zone 为空时，自动从规格的可用区状态中选取第一个可用区

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组及其规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
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
- **delete_default_rules**：是否删除默认规则，设置为 true 以删除默认规则
- **security_group_id**：安全组规则所属的安全组ID，引用前一步创建的安全组的ID进行赋值
- **direction**：规则方向，设置为 ingress 表示入方向
- **ethertype**：IP协议类型，设置为 IPv4
- **remote_ip_prefix**：远端IP地址范围，通过引用输入变量 vpc_cidr 进行赋值
- **ports**：端口范围，通过引用输入变量 instance_db_port 进行赋值
- **protocol**：协议类型，设置为 tcp

### 6. 创建随机密码

在TF文件（如main.tf）中添加以下脚本以创建随机密码：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建随机密码资源
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
- **length**：密码长度，设置为 12
- **special**：是否包含特殊字符，设置为 true
- **override_special**：允许使用的特殊字符集合
- **min_upper**、**min_lower**、**min_numeric**、**min_special**：分别表示大写字母、小写字母、数字和特殊字符的最小数量

### 7. 创建TaurusDB参数模板

在TF文件（如main.tf）中添加以下脚本以创建TaurusDB参数模板：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建TaurusDB参数模板资源
variable "configuration_id" {
  description = "The ID of an existing parameter template. If not specified, a new parameter template will be created"
  type        = string
  default     = ""
}

variable "parameter_template_name" {
  description = "The name of the parameter template to create"
  type        = string
  default     = ""
}

resource "huaweicloud_taurusdb_parameter_template" "test" {
  count = var.configuration_id == "" ? 1 : 0

  name              = var.parameter_template_name
  datastore_engine  = "gaussdb-mysql"
  datastore_version = "8.0"

  parameter_values = {
    auto_increment_increment = "100"
    character_set_server     = "gbk"
  }
}
```

**参数说明**：
- **count**：当输入变量 configuration_id 为空时创建参数模板，否则不创建
- **name**：参数模板名称，通过引用输入变量 parameter_template_name 进行赋值
- **datastore_engine**：数据库引擎，固定为 gaussdb-mysql
- **datastore_version**：数据库版本，固定为 8.0
- **parameter_values**：参数模板中的参数值，示例中设置了自增步长和服务器字符集

### 8. 创建TaurusDB实例

在TF文件（如main.tf）中添加以下脚本以创建TaurusDB实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建TaurusDB实例资源
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
  flavor                           = var.instance_flavor_ref != "" ? var.instance_flavor_ref : try(data.huaweicloud_taurusdb_flavors.test.flavors[0].name, "")
  vpc_id                           = huaweicloud_vpc.test.id
  subnet_id                        = huaweicloud_vpc_subnet.test.id
  security_group_id                = huaweicloud_networking_secgroup.test.id
  password                         = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
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
  configuration_id                 = var.configuration_id != "" ? var.configuration_id : try(huaweicloud_taurusdb_parameter_template.test[0].id, null)
  maintain_begin                   = var.maintain_begin
  maintain_end                     = var.maintain_end
  description                      = var.description
  seconds_level_monitoring_enabled = var.seconds_level_monitoring_enabled
  seconds_level_monitoring_period  = var.seconds_level_monitoring_enabled ? var.seconds_level_monitoring_period : null

  datastore {
    engine  = "gaussdb-mysql"
    version = "8.0"
  }

  audit_log_enabled   = var.audit_log_enabled
  audit_log_keep_days = var.audit_log_keep_days
  reserve_audit_logs  = var.reserve_audit_logs

  backup_strategy {
    start_time = var.instance_backup_time_window
    keep_days  = tostring(var.instance_backup_keep_days)
  }

  tags = var.tags

  lifecycle {
    ignore_changes = [
      password, reserve_audit_logs, ssl_option, datastore[0].version,
    ]
  }
}
```

**参数说明**：
- **name**：TaurusDB实例名称，通过引用输入变量 instance_name 进行赋值
- **flavor**：实例规格，当输入变量 instance_flavor_ref 为空时，自动使用数据源查询到的第一个可用规格
- **vpc_id**：实例所属的VPC ID，引用前一步创建的VPC的ID进行赋值
- **subnet_id**：实例所属的子网ID，引用前一步创建的子网的ID进行赋值
- **security_group_id**：实例所属的安全组ID，引用前一步创建的安全组的ID进行赋值
- **password**：实例密码，当输入变量 instance_password 为空时，使用随机密码
- **mode**：实例模式，通过引用输入变量 instance_mode 进行赋值
- **availability_zone_mode**：可用区模式，通过引用输入变量 availability_zone_mode 进行赋值
- **master_availability_zone**：主节点可用区，引用本地变量 master_az 进行赋值
- **read_replicas**：只读副本数量，通过引用输入变量 read_replicas 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **volume_type**：存储类型，通过引用输入变量 volume_type 进行赋值
- **time_zone**：时区，通过引用输入变量 time_zone 进行赋值
- **port**：数据库端口，通过引用输入变量 instance_db_port 进行赋值
- **ssl_option**：是否启用SSL，通过引用输入变量 ssl_option 进行赋值
- **sql_filter_enabled**：是否启用SQL过滤，通过引用输入变量 sql_filter_enabled 进行赋值
- **slow_log_show_original_switch**：是否启用慢日志显示原始开关，通过引用输入变量 slow_log_show_original_switch 进行赋值
- **table_name_case_sensitivity**：内核表名是否区分大小写，通过引用输入变量 table_name_case_sensitivity 进行赋值
- **multi_tenant_switch**：是否启用多租户开关，通过引用输入变量 multi_tenant_switch 进行赋值
- **configuration_id**：参数模板ID，当输入变量 configuration_id 为空时，使用前一步创建的参数模板的ID
- **maintain_begin**、**maintain_end**：维护窗口的开始和结束时间，通过引用输入变量 maintain_begin 和 maintain_end 进行赋值
- **description**：实例描述，通过引用输入变量 description 进行赋值
- **seconds_level_monitoring_enabled**：是否启用秒级监控，通过引用输入变量 seconds_level_monitoring_enabled 进行赋值
- **seconds_level_monitoring_period**：秒级监控采集周期，通过引用输入变量 seconds_level_monitoring_period 进行赋值
- **datastore**：数据库引擎信息，固定为 gaussdb-mysql 8.0
- **audit_log_enabled**：是否启用审计日志，通过引用输入变量 audit_log_enabled 进行赋值
- **audit_log_keep_days**：审计日志保留天数，通过引用输入变量 audit_log_keep_days 进行赋值
- **reserve_audit_logs**：是否保留历史审计日志，通过引用输入变量 reserve_audit_logs 进行赋值
- **backup_strategy**：备份策略，其中 start_time 通过引用输入变量 instance_backup_time_window 进行赋值，keep_days 通过引用输入变量 instance_backup_keep_days 进行赋值
- **tags**：实例标签，通过引用输入变量 tags 进行赋值

### 9. 创建TaurusDB账号

在TF文件（如main.tf）中添加以下脚本以创建TaurusDB账号：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建TaurusDB账号资源
variable "account_name" {
  description = "Username with elevated privileges"
  type        = string
}

resource "huaweicloud_taurusdb_account" "test" {
  instance_id = huaweicloud_taurusdb_instance.test.id
  name        = var.account_name
  password    = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
}
```

**参数说明**：
- **instance_id**：账号所属的实例ID，引用前一步创建的实例的ID进行赋值
- **name**：账号名称，通过引用输入变量 account_name 进行赋值
- **password**：账号密码，当输入变量 instance_password 为空时，使用随机密码

### 10. 创建TaurusDB数据库

在TF文件（如main.tf）中添加以下脚本以创建TaurusDB数据库：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建TaurusDB数据库资源
variable "database_name" {
  description = "The name of the initial database"
  type        = string
}

variable "character_set" {
  description = "The character set of the database"
  type        = string
  default     = "utf8"
}

resource "huaweicloud_taurusdb_database" "test" {
  instance_id   = huaweicloud_taurusdb_instance.test.id
  name          = var.database_name
  character_set = var.character_set
}
```

**参数说明**：
- **instance_id**：数据库所属的实例ID，引用前一步创建的实例的ID进行赋值
- **name**：数据库名称，通过引用输入变量 database_name 进行赋值
- **character_set**：数据库字符集，通过引用输入变量 character_set 进行赋值

### 11. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name                    = "your_vpc"
subnet_name                 = "your_subnet"
security_group_name         = "your_security_group"
instance_name               = "your_taurusdb_instance"
account_name                = "your_account"
database_name               = "your_database"
instance_backup_time_window = "02:00-03:00"
instance_backup_keep_days   = 7
parameter_template_name     = "your_parameter_template"
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

### 12. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建TaurusDB实例
4. 运行 `terraform show` 查看已创建的TaurusDB实例

## 参考信息

- [华为云TaurusDB产品文档](https://support.huaweicloud.com/taurusdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [TaurusDB实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/taurusdb/taurusdb-instance)
