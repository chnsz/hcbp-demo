# 部署RDS MySQL连接

## 应用场景

数据复制服务（Data Replication Service，DRS）是华为云提供的一站式数据复制服务，用于解决数据库上云、迁移、实时同步和灾备等场景下的数据流转问题。在使用DRS执行迁移、同步或灾备任务之前，需要先通过连接管理功能维护源数据库和目标数据库的接入信息，包括数据库类型、IP地址与端口、用户名与密码、SSL配置以及驱动配置等。

本最佳实践将介绍如何使用Terraform自动化创建一条面向RDS MySQL实例的DRS连接，包括VPC、子网、安全组、RDS MySQL实例以及DRS连接的创建，帮助您快速打通DRS与云上MySQL数据库之间的连接，为后续的数据复制任务奠定基础。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS规格列表（data.huaweicloud_rds_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RDS实例（huaweicloud_rds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DRS连接（huaweicloud_drs_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_connection)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
data.huaweicloud_rds_flavors.test
    └── huaweicloud_rds_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_rds_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_networking_secgroup_rule.test
        └── huaweicloud_rds_instance.test

huaweicloud_rds_instance.test
    └── huaweicloud_drs_connection.test
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
- **cidr**：VPC网段，通过引用输入变量 vpc_cidr 进行赋值

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
- **vpc_id**：子网所属的VPC ID，引用前面创建的VPC资源的ID
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网网段，通过引用输入变量 subnet_cidr 进行赋值；当该变量为空时，基于VPC网段自动划分子网
- **gateway_ip**：子网网关IP，通过引用输入变量 gateway_ip 进行赋值；当该变量为空时，基于子网网段自动计算网关IP

### 4. 创建安全组及其规则

在TF文件（如main.tf）中添加以下脚本以创建安全组及其规则：

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

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = 2

  security_group_id = huaweicloud_networking_secgroup.test.id
  ethertype         = "IPv4"
  remote_ip_prefix  = "192.168.0.0/16"
  protocol          = "tcp"
  direction         = count.index == 0 ? "ingress" : "egress"
  ports             = count.index == 0 ? "3306" : null
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：是否删除安全组默认规则，设置为 true 以便仅保留自定义规则
- **security_group_id**：安全组规则所属的安全组ID，引用前面创建的安全组资源的ID
- **ethertype**：网络类型，设置为 IPv4
- **remote_ip_prefix**：远端IP地址范围，设置为 192.168.0.0/16
- **protocol**：协议类型，设置为 tcp
- **direction**：规则方向，第一条为 ingress（入方向），第二条为 egress（出方向）
- **ports**：端口范围，入方向规则开放 3306 端口用于MySQL访问

### 5. 查询可用区和RDS规格

在TF文件（如main.tf）中添加以下脚本以查询可用区和RDS规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
variable "rds_db_type" {
  description = "The database type for querying RDS flavors"
  type        = string
  default     = "MySQL"
}

variable "rds_db_version" {
  description = "The database version for querying RDS flavors"
  type        = string
  default     = "5.7"
}

variable "rds_instance_mode" {
  description = "The instance mode for querying RDS flavors"
  type        = string
  default     = "single"
}

data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_rds_flavors" "test" {
  db_type       = var.rds_db_type
  db_version    = var.rds_db_version
  instance_mode = var.rds_instance_mode
}
```

**参数说明**：
- **db_type**：数据库类型，通过引用输入变量 rds_db_type 进行赋值
- **db_version**：数据库版本，通过引用输入变量 rds_db_version 进行赋值
- **instance_mode**：实例模式，通过引用输入变量 rds_instance_mode 进行赋值

### 6. 创建RDS MySQL实例

在TF文件（如main.tf）中添加以下脚本以创建RDS MySQL实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS实例资源
variable "rds_name" {
  description = "The name of the RDS instance"
  type        = string
}

variable "rds_flavor" {
  description = "The flavor of the RDS instance. If not specified, it will be queried from data source"
  type        = string
  default     = ""
}

variable "rds_fixed_ip" {
  description = "The fixed IP address of the RDS instance"
  type        = string
  default     = "192.168.0.100"
}

variable "db_password" {
  description = "The password for the RDS root user and DRS connection"
  type        = string
  sensitive   = true
}

resource "huaweicloud_rds_instance" "test" {
  depends_on = [
    huaweicloud_networking_secgroup_rule.test,
  ]

  name              = var.rds_name
  flavor            = var.rds_flavor != "" ? var.rds_flavor : try(data.huaweicloud_rds_flavors.test.flavors[0].name, null)
  security_group_id = huaweicloud_networking_secgroup.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  vpc_id            = huaweicloud_vpc.test.id
  fixed_ip          = var.rds_fixed_ip

  availability_zone = [
    try(data.huaweicloud_availability_zones.test.names[0], ""),
  ]

  db {
    password = var.db_password
    type     = "MySQL"
    version  = "5.7"
    port     = 3306
  }

  volume {
    type = "CLOUDSSD"
    size = 40
  }
}
```

**参数说明**：
- **name**：RDS实例名称，通过引用输入变量 rds_name 进行赋值
- **flavor**：实例规格，通过引用输入变量 rds_flavor 进行赋值；当该变量为空时，从RDS规格数据源中查询获取
- **security_group_id**：实例所属的安全组ID，引用前面创建的安全组资源的ID
- **subnet_id**：实例所属的子网ID，引用前面创建的子网资源的ID
- **vpc_id**：实例所属的VPC ID，引用前面创建的VPC资源的ID
- **fixed_ip**：实例的固定IP地址，通过引用输入变量 rds_fixed_ip 进行赋值
- **availability_zone**：实例所在的可用区，从可用区数据源中查询获取
- **db.password**：数据库密码，通过引用输入变量 db_password 进行赋值
- **db.type**：数据库类型，设置为 MySQL
- **db.version**：数据库版本，设置为 5.7
- **db.port**：数据库端口，设置为 3306
- **volume.type**：磁盘类型，设置为 CLOUDSSD
- **volume.size**：磁盘大小，设置为 40GB

### 7. 创建DRS连接

在TF文件（如main.tf）中添加以下脚本以创建DRS连接：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DRS连接资源
variable "connection_name" {
  description = "The DRS connection name"
  type        = string
}

variable "description" {
  description = "The description of the DRS connection"
  type        = string
  default     = ""
}

variable "db_port" {
  description = "The database port"
  type        = string
  default     = "3306"
}

variable "db_user" {
  description = "The database username"
  type        = string
  default     = "root"
}

variable "driver_name" {
  description = "The driver name of the connection configuration"
  type        = string
  default     = "mysql"
}

resource "huaweicloud_drs_connection" "test" {
  name        = var.connection_name
  db_type     = "mysql"
  description = var.description

  endpoint {
    endpoint_name = "cloud_mysql"
    instance_id   = huaweicloud_rds_instance.test.id
    db_port       = var.db_port
    db_user       = var.db_user
    db_password   = var.db_password
  }

  vpc {
    vpc_id            = huaweicloud_rds_instance.test.vpc_id
    subnet_id         = huaweicloud_rds_instance.test.subnet_id
    security_group_id = huaweicloud_networking_secgroup.test.id
  }

  ssl {
    ssl_link = false
  }

  config {
    driver_name = var.driver_name
  }

  lifecycle {
    ignore_changes = [
      endpoint.0.db_password,
    ]
  }
}
```

**参数说明**：
- **name**：DRS连接名称，通过引用输入变量 connection_name 进行赋值
- **db_type**：数据库类型，设置为 mysql
- **description**：连接描述，通过引用输入变量 description 进行赋值
- **endpoint.endpoint_name**：连接端点名称，设置为 cloud_mysql
- **endpoint.instance_id**：连接端点对应的实例ID，引用前面创建的RDS实例资源的ID
- **endpoint.db_port**：数据库端口，通过引用输入变量 db_port 进行赋值
- **endpoint.db_user**：数据库用户名，通过引用输入变量 db_user 进行赋值
- **endpoint.db_password**：数据库密码，通过引用输入变量 db_password 进行赋值
- **vpc.vpc_id**：连接所属的VPC ID，引用RDS实例的VPC ID
- **vpc.subnet_id**：连接所属的子网ID，引用RDS实例的子网ID
- **vpc.security_group_id**：连接所属的安全组ID，引用前面创建的安全组资源的ID
- **ssl.ssl_link**：是否启用SSL连接，设置为 false
- **config.driver_name**：连接配置的驱动名称，通过引用输入变量 driver_name 进行赋值
- **lifecycle.ignore_changes**：忽略 endpoint.0.db_password 属性的变更，因为该属性不会通过API返回

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# VPC和网络变量
vpc_name            = "your_vpc"
subnet_name         = "your_subnet"
security_group_name = "your_security_group"

# RDS实例变量
rds_name    = "your_rds"
db_password = "TestDrs@123"

# DRS连接变量
connection_name = "your_drs_connection"
description     = "DRS connection for MySQL"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建RDS MySQL连接
4. 运行 `terraform show` 查看已创建的RDS MySQL连接

## 参考信息

- [华为云数据复制服务产品文档](https://support.huaweicloud.com/drs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DRS RDS MySQL连接最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/drs-connection-rds-mysql)
