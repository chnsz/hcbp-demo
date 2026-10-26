# 部署TaurusDB实例并绑定弹性公网IP

## 应用场景

云数据库TaurusDB是华为云提供的企业级云原生数据库服务，默认部署在虚拟私有云（VPC）内，仅支持私网访问。当业务需要从本地数据中心、其他网络环境或公网客户端直接连接数据库时，需要为实例绑定弹性公网IP（EIP），从而获得公网访问入口。

本最佳实践将介绍如何使用Terraform自动化部署TaurusDB实例并为其绑定弹性公网IP，包括VPC与子网创建、安全组及规则配置、实例规格查询、TaurusDB实例创建以及EIP的创建或复用与绑定。脚本支持两种方式：未指定已有EIP地址时自动创建新的EIP并绑定，指定已有EIP地址时查询该EIP并绑定到实例。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [TaurusDB实例规格（data.huaweicloud_taurusdb_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/taurusdb_flavors)
- [弹性公网IP（data.huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/vpc_eip)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [弹性公网IP（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [TaurusDB实例（huaweicloud_taurusdb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_instance)
- [TaurusDB实例绑定EIP（huaweicloud_taurusdb_eip_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/taurusdb_eip_associate)

### 资源/数据源依赖关系

```
data.huaweicloud_taurusdb_flavors.test
    └── huaweicloud_taurusdb_instance.test

data.huaweicloud_vpc_eip.test
    └── huaweicloud_taurusdb_eip_associate.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_taurusdb_instance.test

huaweicloud_networking_secgroup.test
    ├── huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_taurusdb_instance.test

random_password.test
    └── huaweicloud_taurusdb_instance.test

huaweicloud_vpc_eip.test
    └── huaweicloud_taurusdb_eip_associate.test

huaweicloud_taurusdb_instance.test
    └── huaweicloud_taurusdb_eip_associate.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云和子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云和子网资源
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
- **cidr**：VPC的CIDR网段，通过引用输入变量 vpc_cidr 进行赋值
- **vpc_id**：子网所属的VPC ID，引用上一步创建的VPC的ID
- **cidr**（子网）：子网网段，未指定时基于VPC网段自动划分
- **gateway_ip**：子网网关IP，未指定时基于子网网段自动计算

### 3. 创建安全组及安全组规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组及安全组规则资源
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
- **delete_default_rules**：是否删除安全组默认规则，设置为 true
- **security_group_id**：规则所属的安全组ID，引用上一步创建的安全组ID
- **direction**：规则方向，设置为 ingress 表示入方向
- **remote_ip_prefix**：允许访问的源网段，通过引用输入变量 vpc_cidr 进行赋值
- **ports**：允许访问的端口，通过引用输入变量 instance_db_port 进行赋值
- **protocol**：协议类型，设置为 tcp

### 4. 查询TaurusDB实例规格

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询TaurusDB实例规格数据源
variable "availability_zone_mode" {
  description = "The availability zone mode. Valid values are single, multi"
  type        = string
  default     = "multi"
}

data "huaweicloud_taurusdb_flavors" "test" {
  engine                 = "gaussdb-mysql"
  version                = "8.0"
  availability_zone_mode = var.availability_zone_mode
}
```

**参数说明**：
- **engine**：数据库引擎，设置为 gaussdb-mysql
- **version**：数据库版本，设置为 8.0
- **availability_zone_mode**：可用区模式，通过引用输入变量 availability_zone_mode 进行赋值

### 5. 创建或查询弹性公网IP

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建或查询弹性公网IP资源/数据源
variable "associate_eip_address" {
  description = "The existing EIP address to associate with the TaurusDB instance. If not specified, a new EIP will be created"
  type        = string
  default     = ""
}

variable "eip_type" {
  description = "The EIP type"
  type        = string
  default     = "5_bgp"
}

variable "bandwidth_name" {
  description = "The bandwidth name"
  type        = string
  default     = ""
}

variable "bandwidth_size" {
  description = "The bandwidth size in Mbit/s"
  type        = number
  default     = 5
}

variable "bandwidth_share_type" {
  description = "The share type of the bandwidth. Valid values are PER, WHOLE"
  type        = string
  default     = "PER"
}

variable "bandwidth_charge_mode" {
  description = "The charge mode of the bandwidth. Valid values are traffic, bandwidth"
  type        = string
  default     = "traffic"
}

# 指定已有EIP地址时，查询该EIP
data "huaweicloud_vpc_eip" "test" {
  count = var.associate_eip_address != "" ? 1 : 0

  public_ip = var.associate_eip_address
}

# 未指定已有EIP地址时，创建新的EIP
resource "huaweicloud_vpc_eip" "test" {
  count = var.associate_eip_address == "" ? 1 : 0

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.bandwidth_name
    size        = var.bandwidth_size
    share_type  = var.bandwidth_share_type
    charge_mode = var.bandwidth_charge_mode
  }
}
```

**参数说明**：
- **count**：数据源在指定已有EIP地址时创建，资源在未指定已有EIP地址时创建
- **public_ip**：已有EIP的公网地址，通过引用输入变量 associate_eip_address 进行赋值
- **type**：EIP类型，通过引用输入变量 eip_type 进行赋值
- **name**：带宽名称，通过引用输入变量 bandwidth_name 进行赋值
- **size**：带宽大小，通过引用输入变量 bandwidth_size 进行赋值
- **share_type**：带宽共享类型，通过引用输入变量 bandwidth_share_type 进行赋值
- **charge_mode**：带宽计费模式，通过引用输入变量 bandwidth_charge_mode 进行赋值

### 6. 创建TaurusDB实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建TaurusDB实例资源
variable "master_availability_zone" {
  description = "The master availability zone of the TaurusDB instance. If not specified, the first available AZ from flavors will be used"
  type        = string
  default     = ""
}

variable "instance_password" {
  description = "The password for the TaurusDB instance"
  type        = string
  default     = ""
  sensitive   = true
}

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

locals {
  # 从规格的可用区状态中获取第一个可用可用区
  available_azs = try([for k, v in data.huaweicloud_taurusdb_flavors.test.flavors[0].az_status : k if v == "normal"], [])
  master_az     = var.master_availability_zone != "" ? var.master_availability_zone : try(local.available_azs[0], "")
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
- **name**：实例名称，通过引用输入变量 instance_name 进行赋值
- **flavor**：实例规格，未指定时使用规格数据源查询到的第一个可用规格
- **vpc_id**：实例所属的VPC ID，引用前面创建的VPC的ID
- **subnet_id**：实例所属的子网ID，引用前面创建的子网的ID
- **security_group_id**：实例所属的安全组ID，引用前面创建的安全组的ID
- **password**：实例密码，未指定时使用随机密码资源生成
- **mode**：实例模式，通过引用输入变量 instance_mode 进行赋值
- **availability_zone_mode**：可用区模式，通过引用输入变量 availability_zone_mode 进行赋值
- **master_availability_zone**：主可用区，未指定时使用规格数据源查询到的第一个可用可用区
- **read_replicas**：只读节点数量，通过引用输入变量 read_replicas 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **volume_type**：存储类型，通过引用输入变量 volume_type 进行赋值
- **time_zone**：时区，通过引用输入变量 time_zone 进行赋值
- **port**：数据库端口，通过引用输入变量 instance_db_port 进行赋值
- **ssl_option**：是否开启SSL，通过引用输入变量 ssl_option 进行赋值
- **sql_filter_enabled**：是否开启SQL过滤，通过引用输入变量 sql_filter_enabled 进行赋值
- **slow_log_show_original_switch**：是否开启慢日志显示原始语句开关，通过引用输入变量 slow_log_show_original_switch 进行赋值
- **table_name_case_sensitivity**：内核表名是否大小写敏感，通过引用输入变量 table_name_case_sensitivity 进行赋值
- **multi_tenant_switch**：是否开启多租户开关，通过引用输入变量 multi_tenant_switch 进行赋值
- **maintain_begin** / **maintain_end**：可维护时间窗起止时间，通过引用输入变量 maintain_begin 和 maintain_end 进行赋值
- **description**：实例描述，通过引用输入变量 description 进行赋值
- **seconds_level_monitoring_enabled** / **seconds_level_monitoring_period**：秒级监控开关与采集周期，通过引用输入变量进行赋值
- **audit_log_enabled** / **audit_log_keep_days** / **reserve_audit_logs**：审计日志开关、保留天数与历史日志保留策略，通过引用输入变量进行赋值
- **backup_strategy**：备份策略，start_time 引用输入变量 instance_backup_time_window，keep_days 引用输入变量 instance_backup_keep_days
- **tags**：实例标签，通过引用输入变量 tags 进行赋值

### 7. 绑定弹性公网IP到TaurusDB实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建TaurusDB实例绑定EIP资源
locals {
  public_ip    = var.associate_eip_address != "" ? var.associate_eip_address : huaweicloud_vpc_eip.test[0].address
  public_ip_id = var.associate_eip_address != "" ? data.huaweicloud_vpc_eip.test[0].id : huaweicloud_vpc_eip.test[0].id
}

resource "huaweicloud_taurusdb_eip_associate" "test" {
  instance_id  = huaweicloud_taurusdb_instance.test.id
  public_ip    = local.public_ip
  public_ip_id = local.public_ip_id
}
```

**参数说明**：
- **instance_id**：需要绑定EIP的TaurusDB实例ID，引用前面创建的实例的ID
- **public_ip**：绑定的公网IP地址，指定已有EIP地址时使用该地址，否则使用新建EIP的地址
- **public_ip_id**：绑定的公网IP ID，指定已有EIP地址时使用查询到的EIP ID，否则使用新建EIP的ID

### 8. 预设资源部署所需的入参（可选）

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
instance_backup_time_window = "02:00-03:00"
instance_backup_keep_days   = 7
bandwidth_name              = "your_bandwidth"

# 使用已有EIP时填写，此时无需填写bandwidth_name
# associate_eip_address = "your_existing_eip_address"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建TaurusDB实例并绑定弹性公网IP
4. 运行 `terraform show` 查看已创建的TaurusDB实例及绑定关系

## 参考信息

- [华为云TaurusDB产品文档](https://support.huaweicloud.com/taurusdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [TaurusDB实例绑定弹性公网IP最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/taurusdb/taurusdb-instance-associate-eip)
