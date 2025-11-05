# 部署SQL Server单机实例

## 应用场景

关系型数据库服务（Relational Database Service，RDS）是华为云提供的高可用、高性能、易扩展的关系型数据库云服务，支持MySQL、PostgreSQL、SQL Server、MariaDB等多种数据库引擎。RDS提供自动备份、监控告警、弹性扩容、读写分离等功能，能够满足企业级应用的数据库需求。

SQL Server是微软开发的关系型数据库管理系统，广泛应用于企业级应用开发。华为云RDS支持SQL Server数据库引擎，提供SQL Server 2019 SE、2019 EE等版本，支持Windows环境下的数据库应用。

本最佳实践将介绍如何使用Terraform自动化部署一个RDS SQL Server单机实例，包括VPC网络、安全组、RDS实例的创建，支持完整的SQL Server数据库管理功能。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS规格列表查询数据源（data.huaweicloud_rds_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [随机密码资源（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [RDS实例资源（huaweicloud_rds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_vpc_subnet
    └── data.huaweicloud_rds_flavors

data.huaweicloud_rds_flavors
    └── huaweicloud_rds_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet

huaweicloud_vpc_subnet
    └── huaweicloud_rds_instance

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
    └── huaweicloud_rds_instance

huaweicloud_networking_secgroup_rule
    └── huaweicloud_rds_instance

random_password
    └── huaweicloud_rds_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../docs/introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建VPC子网和RDS实例：

```hcl
variable "availability_zone" {
  description = "The availability zone to which the RDS instance belongs"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建VPC子网和RDS实例
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zone` 为空时创建数据源（即执行可用区列表查询）

### 3. 通过数据源查询RDS规格信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建RDS实例：

```hcl
variable "instance_flavor_id" {
  description = "The flavor ID of the RDS instance"
  type        = string
  default     = ""
}

variable "instance_db_type" {
  description = "The database engine type"
  type        = string
  default     = "SQLServer"
}

variable "instance_db_version" {
  description = "The database engine version"
  type        = string
  default     = "2019_SE"
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

variable "instance_flavor_memory" {
  description = "The memory size for the RDS instance flavor"
  type        = number
  default     = 4
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的RDS规格信息，用于创建RDS实例
data "huaweicloud_rds_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  db_type           = var.instance_db_type
  db_version        = var.instance_db_version
  instance_mode     = var.instance_mode
  group_type        = var.instance_flavor_group_type
  vcpus             = var.instance_flavor_vcpus
  memory            = var.instance_flavor_memory
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行RDS规格列表查询数据源，仅当 `var.instance_flavor_id` 为空时创建数据源
- **db_type**：数据库类型，通过引用输入变量 `instance_db_type` 进行赋值
- **db_version**：数据库版本，通过引用输入变量 `instance_db_version` 进行赋值
- **instance_mode**：实例模式，通过引用输入变量 `instance_mode` 进行赋值
- **group_type**：规格组类型，通过引用输入变量 `instance_flavor_group_type` 进行赋值
- **vcpus**：CPU核数，通过引用输入变量 `instance_flavor_vcpus` 进行赋值
- **memory**：内存大小，通过引用输入变量 `instance_flavor_memory` 进行赋值
- **availability_zone**：可用区，根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值

### 4. 创建VPC网络

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
- **name**：VPC的名称，通过引用输入变量 `vpc_name` 进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量 `vpc_cidr` 进行赋值

### 5. 创建VPC子网

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

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
- **vpc_id**：VPC的ID，通过引用VPC资源（huaweicloud_vpc）的ID进行赋值
- **name**：子网的名称，通过引用输入变量 `subnet_name` 进行赋值
- **cidr**：子网的CIDR块，通过引用输入变量 `subnet_cidr` 进行赋值，如果为空则自动计算
- **gateway_ip**：子网的网关IP地址，通过引用输入变量 `gateway_ip` 进行赋值，如果为空则自动计算
- **availability_zone**：子网所在的可用区，根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值

### 6. 创建安全组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

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
- **name**：安全组的名称，通过引用输入变量 `security_group_name` 进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true以删除默认的安全组规则

### 7. 创建安全组规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组规则资源：

```hcl
variable "instance_db_port" {
  description = "The database port"
  type        = number
  default     = 1433
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
- **security_group_id**：安全组的ID，通过引用安全组资源（huaweicloud_networking_secgroup）的ID进行赋值
- **direction**：规则方向，设置为"ingress"表示入站规则
- **ethertype**：以太网类型，设置为"IPv4"
- **remote_ip_prefix**：远程IP前缀，通过引用输入变量 `vpc_cidr` 进行赋值
- **ports**：端口号，通过引用输入变量 `instance_db_port` 进行赋值
- **protocol**：协议类型，设置为"tcp"

### 8. 创建随机密码

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建随机密码资源：

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
- **count**：资源的创建数，用于控制是否执行随机密码资源创建，仅当 `var.instance_password` 为空时创建资源
- **length**：密码长度，设置为12位
- **special**：是否包含特殊字符，设置为true
- **override_special**：特殊字符集合，设置为"!@%^*-_=+"

### 9. 创建RDS SQL Server单机实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建RDS SQL Server单机实例资源：

```hcl
variable "instance_name" {
  description = "The SQLServer RDS instance name"
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
  default     = "03:00-04:00"
}

variable "instance_backup_keep_days" {
  description = "The number of days to retain backups"
  type        = number
  default     = 7
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS SQL Server单机实例资源
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
      availability_zone
    ]
  }

  depends_on = [
    huaweicloud_networking_secgroup_rule.test
  ]
}
```

**参数说明**：
- **name**：RDS实例的名称，通过引用输入变量 `instance_name` 进行赋值
- **flavor**：RDS实例的规格，通过引用输入变量 `instance_flavor_id` 进行赋值，如果为空则根据RDS规格列表查询数据源（data.huaweicloud_rds_flavors）的返回结果进行赋值
- **vpc_id**：VPC的ID，通过引用VPC资源（huaweicloud_vpc）的ID进行赋值
- **subnet_id**：子网的ID，通过引用VPC子网资源（huaweicloud_vpc_subnet）的ID进行赋值
- **security_group_id**：安全组的ID，通过引用安全组资源（huaweicloud_networking_secgroup）的ID进行赋值
- **availability_zone**：可用区列表，根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值
- **db**：数据库配置块，包含数据库类型、版本、端口和密码
- **volume**：存储卷配置块，包含存储类型和大小
- **backup_strategy**：备份策略配置块，包含备份时间窗口和保留天数
- **lifecycle**：生命周期配置块，用于忽略特定参数的变化
- **depends_on**：显式依赖关系，确保安全组规则创建完成后再创建RDS实例

### 10. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 网络配置
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"

# 实例配置
instance_name       = "tf_test_sqlserver_instance"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 11. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建RDS SQL Server单机实例
4. 运行 `terraform show` 查看已创建的RDS SQL Server单机实例

## 参考信息

- [华为云关系型数据库服务产品文档](https://support.huaweicloud.com/rds/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RDS SQL Server单机实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rds/sqlserver-single-instance)
