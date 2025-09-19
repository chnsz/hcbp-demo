# 部署MySQL单机实例

## 应用场景

华为云关系型数据库服务（RDS）的MySQL单机实例功能提供高可用、高性能的MySQL数据库服务，支持自动备份、监控告警、弹性扩容等企业级功能。通过配置MySQL单机实例，您可以快速部署生产级的MySQL数据库，满足Web应用、企业系统、数据分析等场景的数据库需求。

本最佳实践特别适用于需要快速部署MySQL数据库、实现数据持久化存储、构建企业级应用后端的场景，如Web应用开发、企业管理系统、数据分析平台等。本最佳实践将介绍如何使用Terraform自动化部署RDS MySQL单机实例，包括VPC网络、安全组、RDS实例、数据库账户、数据库和备份的创建，实现完整的MySQL数据库管理解决方案。

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
- [RDS MySQL账户资源（huaweicloud_rds_mysql_account）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_mysql_account)
- [RDS MySQL数据库资源（huaweicloud_rds_mysql_database）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_mysql_database)
- [RDS MySQL数据库权限资源（huaweicloud_rds_mysql_database_privilege）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_mysql_database_privilege)
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
    └── huaweicloud_rds_mysql_account

huaweicloud_rds_instance
    ├── huaweicloud_rds_mysql_account
    ├── huaweicloud_rds_mysql_database
    └── huaweicloud_rds_backup

huaweicloud_rds_mysql_account
    └── huaweicloud_rds_mysql_database_privilege

huaweicloud_rds_mysql_database
    └── huaweicloud_rds_mysql_database_privilege

huaweicloud_rds_mysql_database_privilege
    └── huaweicloud_rds_backup
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC网络

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

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
- **name**：VPC的名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"

### 3. 查询可用区信息

在TF文件中添加以下脚本以告知Terraform查询可用区信息：

```hcl
variable "availability_zone" {
  description = "The availability zone to which the RDS instance belongs"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有可用的可用区信息，用于创建RDS实例
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：条件创建，当availability_zone变量为空字符串时创建此数据源

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
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR块，如果subnet_cidr为空则自动计算，否则使用subnet_cidr的值
- **gateway_ip**：子网的网关IP，如果gateway_ip为空则自动计算，否则使用gateway_ip的值
- **availability_zone**：子网所属的可用区，优先使用availability_zone变量，如果为空则使用查询到的第一个可用区

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
  default     = "MySQL"
}

variable "instance_db_version" {
  description = "The database engine version"
  type        = string
  default     = "8.0"
}

variable "instance_mode" {
  description = "The instance mode for the RDS instance flavor"
  type        = string
  default     = "single"
}

variable "instance_flavor_group_type" {
  description = "The group type for the RDS instance flavor"
  type        = string
  default     = "general"
}

variable "instance_flavor_vcpus" {
  description = "The number of the RDS instance CPU cores for the RDS instance flavor"
  type        = number
  default     = 2
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的RDS规格信息，用于创建RDS实例
data "huaweicloud_rds_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  db_type           = var.instance_db_type
  db_version        = var.instance_db_version
  instance_mode     = var.instance_mode
  group_type        = var.instance_flavor_group_type
  vcpus             = var.instance_flavor_vcpus
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
}
```

**参数说明**：
- **count**：条件创建，当instance_flavor_id变量为空字符串时创建此数据源
- **db_type**：数据库引擎类型，通过引用输入变量instance_db_type进行赋值，默认值为"MySQL"
- **db_version**：数据库引擎版本，通过引用输入变量instance_db_version进行赋值，默认值为"8.0"
- **instance_mode**：实例模式，通过引用输入变量instance_mode进行赋值，默认值为"single"
- **group_type**：规格组类型，通过引用输入变量instance_flavor_group_type进行赋值，默认值为"general"
- **vcpus**：CPU核心数，通过引用输入变量instance_flavor_vcpus进行赋值，默认值为2
- **availability_zone**：可用区，优先使用availability_zone变量，如果为空则使用查询到的第一个可用区

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
- **name**：安全组的名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认安全组规则

### 7. 创建安全组规则

在TF文件中添加以下脚本以告知Terraform创建安全组规则资源：

```hcl
variable "instance_db_port" {
  description = "The database port"
  type        = number
  default     = 3306
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
- **ethertype**：IP协议类型，设置为"IPv4"表示IPv4协议
- **remote_ip_prefix**：远程IP前缀，使用VPC的CIDR块
- **ports**：端口号，通过引用输入变量instance_db_port进行赋值，默认值为3306
- **protocol**：协议类型，设置为"tcp"表示TCP协议

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
- **count**：条件创建，当instance_password变量为空字符串时创建此资源
- **length**：密码长度，设置为12个字符
- **special**：是否包含特殊字符，设置为true表示包含特殊字符
- **override_special**：特殊字符集合，设置为"!@%^*-_=+"

### 9. 创建RDS实例

在TF文件中添加以下脚本以告知Terraform创建RDS实例资源：

```hcl
variable "instance_name" {
  description = "The MySQL RDS instance name"
  type        = string
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
  name              = var.instance_name
  flavor            = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_rds_flavors.test[0].flavors[0].name, null)
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  availability_zone = var.availability_zone != "" ? [var.availability_zone] : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), [])

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
    ]
  }
}
```

**参数说明**：
- **name**：RDS实例的名称，通过引用输入变量instance_name进行赋值
- **flavor**：实例规格，优先使用instance_flavor_id变量，如果为空则使用查询到的规格名称
- **vpc_id**：VPC ID，引用前面创建的VPC资源的ID
- **subnet_id**：子网ID，引用前面创建的VPC子网资源的ID
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **availability_zone**：可用区列表，优先使用availability_zone变量，如果为空则使用查询到的可用区
- **db**：数据库配置块
  - **type**：数据库引擎类型，通过引用输入变量instance_db_type进行赋值
  - **version**：数据库引擎版本，通过引用输入变量instance_db_version进行赋值
  - **port**：数据库端口，通过引用输入变量instance_db_port进行赋值
  - **password**：数据库密码，优先使用instance_password变量，如果为空则使用随机生成的密码
- **volume**：存储卷配置块
  - **type**：存储类型，通过引用输入变量instance_volume_type进行赋值，默认值为"CLOUDSSD"
  - **size**：存储大小，通过引用输入变量instance_volume_size进行赋值，默认值为40（GB）
- **backup_strategy**：备份策略配置块
  - **start_time**：备份时间窗口，通过引用输入变量instance_backup_time_window进行赋值
  - **keep_days**：备份保留天数，通过引用输入变量instance_backup_keep_days进行赋值
- **lifecycle.ignore_changes**：生命周期管理，忽略flavor的变更

### 10. 创建RDS MySQL账户

在TF文件中添加以下脚本以告知Terraform创建RDS MySQL账户资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS MySQL账户资源
resource "huaweicloud_rds_mysql_account" "test" {
  instance_id = huaweicloud_rds_instance.test.id
  name        = var.account_name
  password    = var.account_password != "" ? var.account_password : try(random_password.test[0].result, null)
}
```

**参数说明**：
- **instance_id**：RDS实例ID，引用前面创建的RDS实例资源的ID
- **name**：账户名称，通过引用输入变量account_name进行赋值
- **password**：账户密码，优先使用account_password变量，如果为空则使用随机生成的密码

### 11. 创建RDS MySQL数据库

在TF文件中添加以下脚本以告知Terraform创建RDS MySQL数据库资源：

```hcl
variable "database_name" {
  description = "The name of the initial database"
  type        = string
}

variable "character_set" {
  description = "The character set of the database"
  type        = string
  default     = "utf8"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS MySQL数据库资源
resource "huaweicloud_rds_mysql_database" "test" {
  instance_id   = huaweicloud_rds_instance.test.id
  name          = var.database_name
  character_set = var.character_set
}
```

**参数说明**：
- **instance_id**：RDS实例ID，引用前面创建的RDS实例资源的ID
- **name**：数据库名称，通过引用输入变量database_name进行赋值
- **character_set**：字符集，通过引用输入变量character_set进行赋值，默认值为"utf8"

### 12. 创建RDS MySQL数据库权限

在TF文件中添加以下脚本以告知Terraform创建RDS MySQL数据库权限资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS MySQL数据库权限资源
resource "huaweicloud_rds_mysql_database_privilege" "test" {
  instance_id = huaweicloud_rds_instance.test.id
  db_name     = var.database_name

  users {
    name     = huaweicloud_rds_mysql_account.test.name
    readonly = true
  }

  depends_on = [huaweicloud_rds_mysql_database.test]
}
```

**参数说明**：
- **instance_id**：RDS实例ID，引用前面创建的RDS实例资源的ID
- **db_name**：数据库名称，通过引用输入变量database_name进行赋值
- **users**：用户权限配置块
  - **name**：用户名，引用前面创建的RDS MySQL账户资源的名称
  - **readonly**：是否只读权限，设置为true表示只读权限
- **depends_on**：显式依赖关系，确保数据库在权限创建前已存在

### 13. 创建RDS备份

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

  depends_on = [huaweicloud_rds_mysql_database_privilege.test]
}
```

**参数说明**：
- **instance_id**：RDS实例ID，引用前面创建的RDS实例资源的ID
- **name**：备份名称，通过引用输入变量backup_name进行赋值
- **depends_on**：显式依赖关系，确保数据库权限在备份创建前已配置

### 14. 预设资源部署所需的入参（可选）

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
instance_name = "tf_test_mysql_instance"

# 数据库账户配置
account_name = "tf_test_account"

# 数据库配置
database_name = "tf_test_database"

# 备份配置
backup_name                 = "tf_test_backup"
instance_backup_time_window = "08:00-09:00"
instance_backup_keep_days   = 1
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

### 15. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建MySQL单机实例
4. 运行 `terraform show` 查看已创建的MySQL单机实例详情

## 参考信息

- [华为云关系型数据库服务产品文档](https://support.huaweicloud.com/rds/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RDS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rds)
