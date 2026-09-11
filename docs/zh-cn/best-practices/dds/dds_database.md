# 部署数据库角色与用户

## 应用场景

文档数据库服务（Document Database Service，DDS）是华为云提供的高性能、高可靠、高安全性的分布式文档数据库服务，完全兼容MongoDB协议。在实际业务中，除了创建DDS实例外，还需要为数据库创建角色和用户，以便对数据库进行细粒度的访问控制与权限管理。

本最佳实践将介绍如何使用Terraform自动化创建DDS实例的数据库角色与用户，包括VPC、子网、安全组、DDS实例、数据库角色和数据库用户的创建，帮助您快速完成DDS数据库的初始化配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用分区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DDS实例（huaweicloud_dds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)
- [DDS数据库角色（huaweicloud_dds_database_role）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_database_role)
- [DDS数据库用户（huaweicloud_dds_database_user）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_database_user)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dds_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dds_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dds_instance

random_password
    └── huaweicloud_dds_database_user

huaweicloud_dds_instance
    ├── huaweicloud_dds_database_role
    └── huaweicloud_dds_database_user
        └── huaweicloud_dds_database_role
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用分区列表

在TF文件（如main.tf）中添加以下脚本以查询DDS实例可用的可用分区：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用分区列表
variable "availability_zone" {
  description = "The availability zone to which the DDS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：当输入变量 availability_zone 为空时创建该数据源，否则不创建

### 3. 创建VPC

在TF文件（如main.tf）中添加以下脚本以创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC
variable "vpc_name" {
  description = "The name of the VPC"
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
- **name**：通过引用输入变量 vpc_name 进行赋值
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值

### 4. 创建子网

在TF文件（如main.tf）中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建子网
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：通过引用资源 huaweicloud_vpc.test 的 id 进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：当输入变量 subnet_cidr 为空时，基于VPC的CIDR自动划分子网网段，否则引用输入变量 subnet_cidr 进行赋值
- **gateway_ip**：当输入变量 subnet_gateway_ip 为空时，基于子网网段自动计算网关IP，否则引用输入变量 subnet_gateway_ip 进行赋值

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：设置为 true，删除安全组的默认规则

### 6. 创建随机密码

在TF文件（如main.tf）中添加以下脚本以在未指定实例密码时自动生成随机密码：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建随机密码
variable "instance_password" {
  description = "The DDS instance access password"
  type        = string
  sensitive   = true
  default     = ""
}

resource "random_password" "test" {
  count = var.instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "!@#%^*-_+?"
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**参数说明**：
- **count**：当输入变量 instance_password 为空时创建该资源，否则不创建
- **length**：随机密码的长度
- **special**：是否包含特殊字符
- **override_special**：允许使用的特殊字符集合
- **min_upper**：至少包含的大写字母数量
- **min_lower**：至少包含的小写字母数量
- **min_numeric**：至少包含的数字数量
- **min_special**：至少包含的特殊字符数量

### 7. 创建DDS实例

在TF文件（如main.tf）中添加以下脚本以创建DDS实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DDS实例
variable "instance_name" {
  description = "The name of the DDS instance"
  type        = string
}

variable "instance_mode" {
  description = "The type of the DDS instance"
  type        = string
  default     = "ReplicaSet"
}

variable "database_type" {
  description = "The database version type of the DDS instance"
  type        = string
  default     = "DDS-Community"
}

variable "database_version" {
  description = "The database version of the DDS instance"
  type        = string
  default     = "4.0"
}

variable "storage_engine" {
  description = "The storage engine of the DDS instance"
  type        = string
  default     = "wiredTiger"
}

variable "node_type" {
  description = "The type of the DDS instance node"
  type        = string
  default     = "replica"
}

variable "node_number" {
  description = "The number of nodes of the DDS instance"
  type        = number
  default     = 3
}

variable "node_spec_code" {
  description = "The spec code of the DDS instance node"
  type        = string
  default     = "dds.mongodb.s6.large.2.repset"
  nullable    = false
}

variable "node_storage_type" {
  description = "The storage type of the DDS instance node"
  type        = string
  default     = "ULTRAHIGH"
}

variable "node_size" {
  description = "The disk size of the node of the DDS instance"
  type        = number
  default     = 10
}

variable "node_list" {
  description = "The node IDs to be deleted of the DDS instance"
  type        = list(string)
  default     = null
}

resource "huaweicloud_dds_instance" "test" {
  name              = var.instance_name
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  mode              = var.instance_mode
  password          = var.instance_password

  datastore {
    type           = var.database_type
    version        = var.database_version
    storage_engine = var.storage_engine
  }

  flavor {
    type      = var.node_type
    num       = var.node_number
    spec_code = var.node_spec_code
    storage   = var.node_storage_type
    size      = var.node_size
    node_list = var.node_list
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 instance_name 进行赋值
- **availability_zone**：当输入变量 availability_zone 为空时，引用数据源查询到的第一个可用分区，否则引用输入变量 availability_zone 进行赋值
- **vpc_id**：通过引用资源 huaweicloud_vpc.test 的 id 进行赋值
- **subnet_id**：通过引用资源 huaweicloud_vpc_subnet.test 的 id 进行赋值
- **security_group_id**：通过引用资源 huaweicloud_networking_secgroup.test 的 id 进行赋值
- **mode**：通过引用输入变量 instance_mode 进行赋值
- **password**：通过引用输入变量 instance_password 进行赋值
- **datastore.type**：通过引用输入变量 database_type 进行赋值
- **datastore.version**：通过引用输入变量 database_version 进行赋值
- **datastore.storage_engine**：通过引用输入变量 storage_engine 进行赋值
- **flavor.type**：通过引用输入变量 node_type 进行赋值
- **flavor.num**：通过引用输入变量 node_number 进行赋值
- **flavor.spec_code**：通过引用输入变量 node_spec_code 进行赋值
- **flavor.storage**：通过引用输入变量 node_storage_type 进行赋值
- **flavor.size**：通过引用输入变量 node_size 进行赋值
- **flavor.node_list**：通过引用输入变量 node_list 进行赋值

### 8. 创建DDS数据库角色

在TF文件（如main.tf）中添加以下脚本以创建DDS数据库角色：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DDS数据库角色
variable "database_role_name" {
  description = "The database role name"
  type        = string
}

resource "huaweicloud_dds_database_role" "test" {
  instance_id = huaweicloud_dds_instance.test.id
  name        = var.database_role_name
  db_name     = "admin"
}
```

**参数说明**：
- **instance_id**：通过引用资源 huaweicloud_dds_instance.test 的 id 进行赋值
- **name**：通过引用输入变量 database_role_name 进行赋值
- **db_name**：数据库角色所属的数据库名称，固定为 admin

### 9. 创建DDS数据库用户

在TF文件（如main.tf）中添加以下脚本以创建DDS数据库用户：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DDS数据库用户
variable "database_user_name" {
  description = "The database user name"
  type        = string
}

resource "huaweicloud_dds_database_user" "test" {
  instance_id = huaweicloud_dds_instance.test.id
  name        = var.database_user_name
  password    = var.instance_password == "" ? try(random_password.test[0].result, null) : var.instance_password
  db_name     = "admin"

  roles {
    name    = huaweicloud_dds_database_role.test.name
    db_name = "admin"
  }
}
```

**参数说明**：
- **instance_id**：通过引用资源 huaweicloud_dds_instance.test 的 id 进行赋值
- **name**：通过引用输入变量 database_user_name 进行赋值
- **password**：当输入变量 instance_password 为空时，引用随机密码资源生成的密码，否则引用输入变量 instance_password 进行赋值
- **db_name**：数据库用户所属的数据库名称，固定为 admin
- **roles.name**：通过引用资源 huaweicloud_dds_database_role.test 的 name 进行赋值
- **roles.db_name**：数据库角色所属的数据库名称，固定为 admin

### 10. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name            = "tf_test_database"
subnet_name         = "tf_test_database"
security_group_name = "tf_test_database"
instance_name       = "tf_test_database"
database_role_name  = "tf_test_database"
database_user_name  = "tf_test_database"
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

### 11. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DDS数据库角色与用户
4. 运行 `terraform show` 查看已创建的DDS数据库角色与用户

## 参考信息

- [华为云文档数据库服务产品文档](https://support.huaweicloud.com/dds/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DDS数据库角色与用户最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/dds-database)
