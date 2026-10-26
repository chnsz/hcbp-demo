# 部署GeminiDB InfluxDB实例

## 应用场景

GeminiDB InfluxDB是华为云提供的时序数据库服务，兼容InfluxDB 1.8接口，适用于物联网、监控、日志分析等海量时序数据的高并发写入与查询场景。本实践面向需要快速构建高性能时序数据存储底座的业务，通过Terraform自动化完成网络、安全组、实例与备份的创建。

本最佳实践将介绍如何使用Terraform自动化部署GeminiDB InfluxDB实例，包括VPC与子网创建、可用区与规格查询、安全组配置、随机密码生成、实例创建以及实例备份。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [GeminiDB规格列表（data.huaweicloud_gaussdb_nosql_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_nosql_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GeminiDB实例（huaweicloud_geminidb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/geminidb_instance)
- [GeminiDB备份（huaweicloud_geminidb_backup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/geminidb_backup)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_gaussdb_nosql_flavors
        └── huaweicloud_geminidb_instance
            └── huaweicloud_geminidb_backup

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_geminidb_instance

huaweicloud_networking_secgroup
    └── huaweicloud_geminidb_instance

random_password
    └── huaweicloud_geminidb_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC与子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC与子网资源
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
- **cidr**：VPC的CIDR网段，通过引用输入变量vpc_cidr进行赋值，默认值为192.168.0.0/16
- **vpc_id**：子网所属的VPC ID，引用VPC资源的ID进行赋值
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR网段，当输入变量subnet_cidr为空时，基于VPC网段自动划分，否则使用输入变量subnet_cidr进行赋值
- **gateway_ip**：子网网关IP，当输入变量gateway_ip为空时，基于子网网段自动计算，否则使用输入变量gateway_ip进行赋值

### 3. 查询可用区与GeminiDB规格

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区与GeminiDB规格
variable "vcpus" {
  description = "The number of vCPUs of the flavor to query"
  type        = number
  default     = 4
}

variable "availability_zone" {
  description = "The availability zone of the GeminiDB InfluxDB instance. If not specified, the first AZ from data source will be used"
  type        = string
  default     = ""
}

data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_gaussdb_nosql_flavors" "test" {
  vcpus             = var.vcpus
  engine            = "influxdb"
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**参数说明**：
- **vcpus**：查询规格的vCPU数量，通过引用输入变量vcpus进行赋值，默认值为4
- **engine**：数据库引擎类型，固定为influxdb
- **availability_zone**：可用区，当输入变量availability_zone为空时取可用区列表中的第一个可用区，否则使用输入变量availability_zone进行赋值

### 4. 创建安全组

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
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
- **delete_default_rules**：是否删除安全组默认规则，设置为true

### 5. 生成实例随机密码

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下生成实例随机密码
variable "instance_password" {
  description = "The password for the GeminiDB InfluxDB instance"
  type        = string
  default     = ""
  sensitive   = true
}

resource "random_password" "test" {
  count = var.instance_password == "" ? 1 : 0

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
- **count**：当输入变量instance_password为空时创建随机密码，否则不创建
- **length**：密码长度，设置为16
- **special**：是否包含特殊字符，设置为true
- **override_special**：允许使用的特殊字符集合
- **min_upper**、**min_lower**、**min_numeric**、**min_special**：密码中大写字母、小写字母、数字和特殊字符的最小数量

### 6. 创建GeminiDB InfluxDB实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GeminiDB InfluxDB实例资源
variable "instance_name" {
  description = "The GeminiDB InfluxDB instance name"
  type        = string
}

variable "instance_mode" {
  description = "The instance mode. Valid values are Cluster, EnhancedCluster, InfluxdbSingle"
  type        = string
  default     = "InfluxdbSingle"
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = "0"
}

variable "instance_ssl_option" {
  description = "Whether to enable SSL. Valid values are on, off"
  type        = string
  default     = "on"
}

variable "maintenance_start_time" {
  description = "The start time of the maintenance window in UTC HH:MM format"
  type        = string
  default     = "02:00"
}

variable "maintenance_end_time" {
  description = "The end time of the maintenance window in UTC HH:MM format, 4 hours after start time"
  type        = string
  default     = "06:00"
}

variable "instance_flavor_num" {
  description = "The node quantity of the GeminiDB InfluxDB instance"
  type        = number
  default     = 1
}

variable "instance_flavor_size" {
  description = "The disk size in GB of the GeminiDB InfluxDB instance"
  type        = number
  default     = 100
}

variable "instance_flavor_storage" {
  description = "The disk type of the GeminiDB InfluxDB instance"
  type        = string
  default     = "ULTRAHIGH"
}

variable "instance_flavor_spec_code" {
  description = "The resource specification code. If not specified, the first available flavor will be used"
  type        = string
  default     = ""
}

variable "instance_backup_time_window" {
  description = "The backup time window in HH:MM-HH:MM format"
  type        = string
}

variable "instance_backup_keep_days" {
  description = "The number of days to retain backups"
  type        = number
}

variable "cold_storage_size" {
  description = "The size of the cold storage in GB, 0 means no cold storage"
  type        = number
  default     = 0
}

variable "charging_mode" {
  description = "The charging mode. Valid values are prePaid, postPaid"
  type        = string
  default     = "prePaid"
}

variable "period_unit" {
  description = "The charging period unit. Valid values are month, year"
  type        = string
  default     = "month"
}

variable "auto_renew" {
  description = "Whether to enable auto-renew. Valid values are true, false"
  type        = string
  default     = "true"
}

variable "period" {
  description = "The charging period"
  type        = number
  default     = 1
}

variable "tags" {
  description = "The tags of the GeminiDB InfluxDB instance"
  type        = map(string)
  default     = {
    foo = "bar"
    key = "value"
  }
}

resource "huaweicloud_geminidb_instance" "test" {
  name                  = var.instance_name
  availability_zone     = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  password              = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  mode                  = var.instance_mode
  enterprise_project_id = var.enterprise_project_id
  ssl_option            = var.instance_ssl_option

  maintenance_start_time = var.maintenance_start_time
  maintenance_end_time   = var.maintenance_end_time

  datastore {
    type           = "influxdb"
    version        = "1.8"
    storage_engine = "rocksDB"
  }

  flavor {
    num       = var.instance_flavor_num
    size      = var.instance_flavor_size
    storage   = var.instance_flavor_storage
    spec_code = var.instance_flavor_spec_code != "" ? var.instance_flavor_spec_code : try(data.huaweicloud_gaussdb_nosql_flavors.test.flavors[0].name, null)
  }

  backup_strategy {
    start_time = var.instance_backup_time_window
    keep_days  = var.instance_backup_keep_days
  }

  cold_storage_size = var.cold_storage_size

  charging_mode = var.charging_mode
  period_unit   = var.period_unit
  auto_renew    = var.auto_renew
  period        = var.period

  tags = var.tags

  lifecycle {
    ignore_changes = [
      password,
      flavor.0.storage,
      ssl_option,
      bucket_name,
      data_export_switch,
      cold_storage_size,
      auto_renew,
      period,
      period_unit,
    ]
  }
}
```

**参数说明**：
- **name**：实例名称，通过引用输入变量instance_name进行赋值
- **availability_zone**：可用区，当输入变量availability_zone为空时取可用区列表中的第一个可用区，否则使用输入变量availability_zone进行赋值
- **vpc_id**：实例所属的VPC ID，引用VPC资源的ID进行赋值
- **subnet_id**：实例所属的子网ID，引用子网资源的ID进行赋值
- **security_group_id**：实例所属的安全组ID，引用安全组资源的ID进行赋值
- **password**：实例密码，当输入变量instance_password不为空时使用该值，否则使用随机密码资源生成的结果
- **mode**：实例模式，通过引用输入变量instance_mode进行赋值，默认值为InfluxdbSingle
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为0
- **ssl_option**：是否开启SSL，通过引用输入变量instance_ssl_option进行赋值，默认值为on
- **maintenance_start_time**、**maintenance_end_time**：可维护时间窗口的开始与结束时间，分别通过引用输入变量maintenance_start_time和maintenance_end_time进行赋值
- **datastore.type**：数据库引擎类型，固定为influxdb
- **datastore.version**：数据库引擎版本，固定为1.8
- **datastore.storage_engine**：存储引擎，固定为rocksDB
- **flavor.num**：节点数量，通过引用输入变量instance_flavor_num进行赋值
- **flavor.size**：磁盘大小（GB），通过引用输入变量instance_flavor_size进行赋值
- **flavor.storage**：磁盘类型，通过引用输入变量instance_flavor_storage进行赋值
- **flavor.spec_code**：资源规格编码，当输入变量instance_flavor_spec_code不为空时使用该值，否则使用规格查询数据源返回的第一个规格名称
- **backup_strategy.start_time**：备份时间窗口，通过引用输入变量instance_backup_time_window进行赋值
- **backup_strategy.keep_days**：备份保留天数，通过引用输入变量instance_backup_keep_days进行赋值
- **cold_storage_size**：冷存储大小（GB），通过引用输入变量cold_storage_size进行赋值，0表示不启用冷存储
- **charging_mode**：计费模式，通过引用输入变量charging_mode进行赋值，默认值为prePaid
- **period_unit**：计费周期单位，通过引用输入变量period_unit进行赋值，默认值为month
- **auto_renew**：是否自动续费，通过引用输入变量auto_renew进行赋值，默认值为true
- **period**：计费周期，通过引用输入变量period进行赋值，默认值为1
- **tags**：实例标签，通过引用输入变量tags进行赋值
- **lifecycle.ignore_changes**：忽略创建后可能发生漂移的属性，包括password、flavor.0.storage、ssl_option、bucket_name、data_export_switch、cold_storage_size、auto_renew、period和period_unit

### 7. 创建GeminiDB备份

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GeminiDB备份资源
variable "backup_name" {
  description = "The name of the GeminiDB backup"
  type        = string
  default     = "tf_test_backup"
}

variable "backup_description" {
  description = "The description of the GeminiDB backup"
  type        = string
  default     = "Created by Terraform"
}

resource "huaweicloud_geminidb_backup" "test" {
  instance_id = huaweicloud_geminidb_instance.test.id
  name        = var.backup_name
  description = var.backup_description

  depends_on = [huaweicloud_geminidb_instance.test]
}
```

**参数说明**：
- **instance_id**：备份所属的实例ID，引用GeminiDB实例资源的ID进行赋值
- **name**：备份名称，通过引用输入变量backup_name进行赋值
- **description**：备份描述，通过引用输入变量backup_description进行赋值
- **depends_on**：显式声明依赖关系，确保实例创建完成后再创建备份

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name                    = "your_vpc1"
subnet_name                 = "your_subnet"
security_group_name         = "your_security_group"
instance_name               = "your_geminidb_influxdb_instance"
instance_backup_time_window = "03:00-04:00"
instance_backup_keep_days   = 14
tags                        = {
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

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建GeminiDB InfluxDB实例
4. 运行 `terraform show` 查看已创建的GeminiDB InfluxDB实例

## 参考信息

- [华为云GeminiDB产品文档](https://support.huaweicloud.com/geminidb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GeminiDB InfluxDB实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/geminidb/geminidb-influxdb-instance)
