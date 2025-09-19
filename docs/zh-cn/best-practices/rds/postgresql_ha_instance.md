# 部署PostgreSQL主备实例

## 应用场景

华为云关系型数据库服务（RDS）的PostgreSQL主备实例功能提供高可用、高性能的PostgreSQL数据库服务，支持主备架构、自动故障切换、读写分离等企业级功能。通过配置PostgreSQL主备实例，您可以构建高可用的数据库集群，满足生产环境对数据安全性和服务连续性的严格要求。

本最佳实践特别适用于需要高可用数据库服务、实现数据冗余备份、构建企业级应用后端的场景，如生产系统数据库、关键业务应用、数据仓库等。本最佳实践将介绍如何使用Terraform自动化部署RDS PostgreSQL主备实例，包括VPC网络、安全组、RDS实例、PostgreSQL账户、数据库、Schema和备份的创建，实现完整的PostgreSQL高可用数据库解决方案。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS规格查询数据源（data.huaweicloud_rds_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [随机密码资源（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [RDS实例资源（huaweicloud_rds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [RDS PostgreSQL账户资源（huaweicloud_rds_pg_account）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_pg_account)
- [RDS PostgreSQL账户权限资源（huaweicloud_rds_pg_account_privileges）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_pg_account_privileges)
- [RDS PostgreSQL数据库资源（huaweicloud_rds_pg_database）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_pg_database)
- [RDS PostgreSQL Schema资源（huaweicloud_rds_pg_schema）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_pg_schema)
- [RDS备份资源（huaweicloud_rds_backup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_backup)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_rds_instance

data.huaweicloud_rds_flavors
    └── huaweicloud_rds_instance

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_rds_instance

huaweicloud_vpc_subnet
    └── huaweicloud_rds_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_rds_instance

huaweicloud_networking_secgroup_rule
    └── huaweicloud_rds_instance

random_password
    ├── huaweicloud_rds_instance
    └── huaweicloud_rds_pg_account

huaweicloud_rds_instance
    ├── huaweicloud_rds_pg_account
    ├── huaweicloud_rds_pg_database
    └── huaweicloud_rds_backup

huaweicloud_rds_pg_account
    ├── huaweicloud_rds_pg_account_privileges
    └── huaweicloud_rds_pg_schema

huaweicloud_rds_pg_database
    └── huaweicloud_rds_pg_schema

huaweicloud_rds_pg_schema
    └── huaweicloud_rds_backup
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC资源

在TF文件中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"

### 3. 查询可用区信息

在TF文件中添加以下脚本以告知Terraform查询可用区信息：

```hcl
variable "availability_zones" {
  description = "The list of availability zones to which the RDS instance belong"
  type        = list(string)
  default     = []
  nullable    = false

  validation {
    condition     = var.instance_mode == "ha" && (length(var.availability_zones) == 0 || length(var.availability_zones) > 1) || var.instance_mode != "ha" && length(var.availability_zones) <= 1
    error_message = "The availability zones must be a list of strings"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区信息
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) < 1 ? 1 : 0
}
```

**参数说明**：
- **count**：当availability_zones变量为空时查询可用区信息，否则不查询
- **availability_zones**：可用区列表，通过引用输入变量availability_zones进行赋值，支持验证规则确保主备模式需要多个可用区

### 4. 创建VPC子网

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  name              = var.subnet_name
  cidr              = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip        = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
  availability_zone = length(var.availability_zones) > 0 ? element(var.availability_zones, 0) : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**参数说明**：
- **vpc_id**：VPC ID，引用前面创建的VPC资源的ID
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网CIDR块，优先使用subnet_cidr变量，如果为空则自动计算
- **gateway_ip**：网关IP地址，优先使用gateway_ip变量，如果为空则自动计算
- **availability_zone**：可用区，优先使用availability_zones变量的第一个元素，否则使用查询到的第一个可用区

### 5. 查询RDS规格信息

在TF文件中添加以下脚本以告知Terraform查询RDS规格信息：

```hcl
variable "instance_flavor_id" {
  description = "The flavor ID of the RDS instance"
  type        = string
  default     = ""
}

variable "instance_db_type" {
  description = "The database engine type"
  type        = string
  default     = "PostgreSQL"
}

variable "instance_db_version" {
  description = "The database engine version"
  type        = string
  default     = "16"
}

variable "instance_mode" {
  description = "The instance mode for the RDS instance flavor"
  type        = string
  default     = "ha"
}

variable "instance_flavor_group_type" {
  description = "The group type for the RDS instance flavor"
  type        = string
  default     = "general"
}

variable "instance_flavor_vcpus" {
  description = "The CPU core numbers for the RDS instance flavor"
  type        = number
  default     = 4
}

variable "instance_flavor_memory" {
  description = "The memory size for the RDS instance flavor"
  type        = number
  default     = 8
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询RDS规格信息
data "huaweicloud_rds_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  db_type           = var.instance_db_type
  db_version        = var.instance_db_version
  instance_mode     = var.instance_mode
  group_type        = var.instance_flavor_group_type
  vcpus             = var.instance_flavor_vcpus
  memory            = var.instance_flavor_memory
  availability_zone = length(var.availability_zones) > 0 ? element(var.availability_zones, 0) : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**参数说明**：
- **count**：当instance_flavor_id变量为空时查询RDS规格信息，否则不查询
- **db_type**：数据库引擎类型，通过引用输入变量instance_db_type进行赋值，默认值为"PostgreSQL"
- **db_version**：数据库引擎版本，通过引用输入变量instance_db_version进行赋值，默认值为"16"
- **instance_mode**：实例模式，通过引用输入变量instance_mode进行赋值，默认值为"ha"（主备模式）
- **group_type**：规格组类型，通过引用输入变量instance_flavor_group_type进行赋值，默认值为"general"
- **vcpus**：CPU核数，通过引用输入变量instance_flavor_vcpus进行赋值，默认值为4
- **memory**：内存大小，通过引用输入变量instance_flavor_memory进行赋值，默认值为8
- **availability_zone**：可用区，优先使用availability_zones变量的第一个元素，否则使用查询到的第一个可用区

### 6. 创建安全组

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true以删除默认的安全组规则

### 7. 创建安全组规则

在TF文件中添加以下脚本以告知Terraform创建安全组规则资源：

```hcl
variable "instance_db_port" {
  description = "The database port"
  type        = number
  default     = 5432
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组规则资源
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
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **direction**：规则方向，设置为"ingress"表示入站规则
- **ethertype**：以太网类型，设置为"IPv4"
- **remote_ip_prefix**：远程IP前缀，通过引用输入变量vpc_cidr进行赋值，允许VPC内访问
- **ports**：端口号，通过引用输入变量instance_db_port进行赋值，默认值为5432（PostgreSQL默认端口）
- **protocol**：协议类型，设置为"tcp"

### 8. 创建随机密码

在TF文件中添加以下脚本以告知Terraform创建随机密码资源：

```hcl
variable "instance_password" {
  description = "The password for the RDS instance"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建随机密码资源
resource "random_password" "test" {
  count = var.instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "!@%^*-_=+"
}
```

**参数说明**：
- **count**：当instance_password变量为空时创建随机密码，否则不创建
- **length**：密码长度，设置为12位
- **special**：是否包含特殊字符，设置为true
- **override_special**：特殊字符集，设置为"!@%^*-_=+"

### 9. 创建RDS实例

在TF文件中添加以下脚本以告知Terraform创建RDS实例资源：

```hcl
variable "instance_name" {
  description = "The name of the RDS instance"
  type        = string
}

variable "ha_replication_mode" {
  description = "The HA replication mode of the RDS instance"
  type        = string
  default     = "async"
}

variable "instance_volume_type" {
  description = "The storage volume type"
  type        = string
  default     = "CLOUDSSD"
}

variable "instance_volume_size" {
  description = "The storage volume size in GB"
  type        = number
  default     = 40
}

variable "instance_backup_time_window" {
  description = "The backup time window in HH:MM-HH:MM format"
  type        = string
}

variable "instance_backup_keep_days" {
  description = "The number of days to retain backups"
  type        = number
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS实例资源
resource "huaweicloud_rds_instance" "test" {
  name                = var.instance_name
  flavor              = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_rds_flavors.test[0].flavors[0].name, null)
  vpc_id              = huaweicloud_vpc.test.id
  subnet_id           = huaweicloud_vpc_subnet.test.id
  security_group_id   = huaweicloud_networking_secgroup.test.id
  availability_zone   = length(var.availability_zones) > 0 ? var.availability_zones : var.instance_mode == "ha" ? try(
    slice(data.huaweicloud_availability_zones.test[0].names, 0, 2), []) : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), [])
  ha_replication_mode = var.instance_mode == "ha" ? var.ha_replication_mode : null

  db {
    type     = var.instance_db_type
    version  = var.instance_db_version
    port     = var.instance_db_port
    password = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  }

  volume {
    type = var.instance_volume_type
    size = var.instance_volume_size
  }

  backup_strategy {
    start_time = var.instance_backup_time_window
    keep_days  = var.instance_backup_keep_days
  }

  lifecycle {
    ignore_changes = [
      flavor,
      availability_zone,
    ]
  }
}
```

**参数说明**：
- **name**：RDS实例名称，通过引用输入变量instance_name进行赋值
- **flavor**：实例规格，优先使用instance_flavor_id变量，如果为空则使用查询到的第一个规格
- **vpc_id**：VPC ID，引用前面创建的VPC资源的ID
- **subnet_id**：子网ID，引用前面创建的子网资源的ID
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **availability_zone**：可用区列表，主备模式需要多个可用区，单机模式只需要一个可用区
- **ha_replication_mode**：主备复制模式，通过引用输入变量ha_replication_mode进行赋值，默认值为"async"
- **db**：数据库配置块，包含数据库类型、版本、端口和密码
- **volume**：存储配置块，包含存储类型和大小
- **backup_strategy**：备份策略配置块，包含备份时间窗口和保留天数
- **lifecycle**：生命周期配置，忽略flavor和availability_zone的变更

### 10. 创建RDS PostgreSQL账户

在TF文件中添加以下脚本以告知Terraform创建RDS PostgreSQL账户资源：

```hcl
variable "account_name" {
  description = "Username with elevated privileges"
  type        = string
}

variable "account_password" {
  description = "The password for the database account"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS PostgreSQL账户资源
resource "huaweicloud_rds_pg_account" "test" {
  instance_id = huaweicloud_rds_instance.test.id
  name        = var.account_name
  password    = var.account_password != "" ? var.account_password : try(random_password.test[0].result, null)
}
```

**参数说明**：
- **instance_id**：RDS实例ID，引用前面创建的RDS实例资源的ID
- **name**：账户名称，通过引用输入变量account_name进行赋值
- **password**：账户密码，优先使用account_password变量，如果为空则使用随机生成的密码

### 11. 创建RDS PostgreSQL账户权限

在TF文件中添加以下脚本以告知Terraform创建RDS PostgreSQL账户权限资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS PostgreSQL账户权限资源
resource "huaweicloud_rds_pg_account_privileges" "test" {
  instance_id            = huaweicloud_rds_instance.test.id
  user_name              = huaweicloud_rds_pg_account.test.name
  role_privileges        = ["CREATEROLE", "CREATEDB", "LOGIN", "REPLICATION"]
  system_role_privileges = ["pg_signal_backend"]
}
```

**参数说明**：
- **instance_id**：RDS实例ID，引用前面创建的RDS实例资源的ID
- **user_name**：用户名，引用前面创建的RDS PostgreSQL账户资源的名称
- **role_privileges**：角色权限列表，设置为["CREATEROLE", "CREATEDB", "LOGIN", "REPLICATION"]
- **system_role_privileges**：系统角色权限列表，设置为["pg_signal_backend"]

### 12. 创建RDS PostgreSQL数据库

在TF文件中添加以下脚本以告知Terraform创建RDS PostgreSQL数据库资源：

```hcl
variable "database_name" {
  description = "The name of the initial database"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS PostgreSQL数据库资源
resource "huaweicloud_rds_pg_database" "test" {
  instance_id = huaweicloud_rds_instance.test.id
  name        = var.database_name
}
```

**参数说明**：
- **instance_id**：RDS实例ID，引用前面创建的RDS实例资源的ID
- **name**：数据库名称，通过引用输入变量database_name进行赋值

### 13. 创建RDS PostgreSQL Schema

在TF文件中添加以下脚本以告知Terraform创建RDS PostgreSQL Schema资源：

```hcl
variable "schema_name" {
  description = "The name of the database schema"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS PostgreSQL Schema资源
resource "huaweicloud_rds_pg_schema" "test" {
  instance_id = huaweicloud_rds_instance.test.id
  db_name     = huaweicloud_rds_pg_database.test.name
  owner       = huaweicloud_rds_pg_account.test.name
  schema_name = var.schema_name
}
```

**参数说明**：
- **instance_id**：RDS实例ID，引用前面创建的RDS实例资源的ID
- **db_name**：数据库名称，引用前面创建的RDS PostgreSQL数据库资源的名称
- **owner**：Schema所有者，引用前面创建的RDS PostgreSQL账户资源的名称
- **schema_name**：Schema名称，通过引用输入变量schema_name进行赋值

### 14. 创建RDS备份

在TF文件中添加以下脚本以告知Terraform创建RDS备份资源：

```hcl
variable "backup_name" {
  description = "The name for instance backups"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS备份资源
resource "huaweicloud_rds_backup" "test" {
  instance_id = huaweicloud_rds_instance.test.id
  name        = var.backup_name

  depends_on = [huaweicloud_rds_pg_schema.test]
}
```

**参数说明**：
- **instance_id**：RDS实例ID，引用前面创建的RDS实例资源的ID
- **name**：备份名称，通过引用输入变量backup_name进行赋值
- **depends_on**：显式依赖关系，确保Schema在备份创建前已配置

### 15. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC网络配置
vpc_name = "tf_test_vpc"

# 子网配置
subnet_name = "tf_test_subnet"

# 安全组配置
security_group_name = "tf_test_security_group"

# RDS实例配置
instance_name = "tf_test_postgresql_instance"

# 备份配置
instance_backup_time_window = "08:00-09:00"
instance_backup_keep_days   = 1

# 数据库账户配置
account_name = "tf_test_account"

# 数据库配置
database_name = "tf_test_database"

# Schema配置
schema_name = "tf_test_schema"

# 备份配置
backup_name = "tf_test_backup"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="instance_name=my-instance"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 16. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建PostgreSQL主备实例
4. 运行 `terraform show` 查看已创建的PostgreSQL主备实例详情

## 参考信息

- [华为云关系型数据库服务产品文档](https://support.huaweicloud.com/rds/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RDS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rds)
