# 部署Redis会话清理

## 应用场景

分布式缓存服务（DCS）Redis实例在运行过程中会与多个客户端建立连接，当某些客户端出现异常或连接泄漏时，会占用实例的连接资源，影响实例的稳定运行。通过会话清理功能，可以按客户端地址批量断开指定会话，及时释放连接资源。

本最佳实践将介绍如何使用Terraform自动化部署一个DCS Redis单机实例，并基于实例分片信息执行会话清理操作，包括VPC创建、子网配置、可用分区与产品规格查询、实例配置、分片查询和会话清理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用分区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS产品规格列表（data.huaweicloud_dcs_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)
- [DCS实例分片列表（data.huaweicloud_dcs_instance_shards）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_instance_shards)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DCS实例（huaweicloud_dcs_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS会话清理（huaweicloud_dcs_sessions_kill）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_sessions_kill)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
            └── data.huaweicloud_dcs_instance_shards
                └── huaweicloud_dcs_sessions_kill

random_password
    └── huaweicloud_dcs_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云资源
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

### 3. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网资源
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
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = (var.subnet_gateway_ip != "" ? var.subnet_gateway_ip :
  cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1))
}
```

**参数说明**：
- **vpc_id**：通过引用资源 huaweicloud_vpc.test 的 ID 进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：当输入变量 subnet_cidr 为空时，基于VPC的CIDR块自动划分子网网段
- **gateway_ip**：当输入变量 subnet_gateway_ip 为空时，自动取子网网段的第一个可用IP作为网关地址

### 4. 查询可用分区列表

在TF文件（如main.tf）中添加以下脚本以查询可用分区列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用分区列表
variable "availability_zone" {
  description = "The availability zone to which the Redis single instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：当输入变量 availability_zone 为空时执行查询，否则不执行

### 5. 查询DCS产品规格列表

在TF文件（如main.tf）中添加以下脚本以查询DCS产品规格列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DCS产品规格列表
variable "instance_flavor_id" {
  description = "The flavor ID of the Redis single instance"
  type        = string
  default     = ""
}

variable "instance_capacity" {
  description = "The capacity of the Redis instance (in GB)"
  type        = number
  default     = 1
}

variable "instance_engine_version" {
  description = "The engine version of the Redis single instance"
  type        = string
  default     = "7.0"
}

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  cache_mode     = "single"
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**参数说明**：
- **count**：当输入变量 instance_flavor_id 为空时执行查询，否则不执行
- **cache_mode**：缓存模式，固定为 single
- **capacity**：通过引用输入变量 instance_capacity 进行赋值
- **engine_version**：通过引用输入变量 instance_engine_version 进行赋值

### 6. 创建随机密码

在TF文件（如main.tf）中添加以下脚本以创建随机密码：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建随机密码资源
variable "instance_password" {
  description = "The password for the Redis instance"
  type        = string
  sensitive   = true
  default     = null
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
- **length**：密码长度，固定为 12
- **special**：是否包含特殊字符，固定为 true
- **override_special**：允许使用的特殊字符集合
- **min_upper**、**min_lower**、**min_numeric**、**min_special**：分别指定大写字母、小写字母、数字和特殊字符的最小数量

### 7. 创建DCS实例

在TF文件（如main.tf）中添加以下脚本以创建DCS Redis单机实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DCS实例资源
variable "instance_name" {
  description = "The name of the Redis single instance"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the Redis single instance belongs"
  type        = string
  default     = null
}

variable "charging_mode" {
  description = "The charging mode of the Redis instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The unit of the period"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the Redis instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "Whether auto renew is enabled"
  type        = string
  default     = "false"
}

resource "huaweicloud_dcs_instance" "test" {
  name                  = var.instance_name
  engine                = "Redis"
  enterprise_project_id = var.enterprise_project_id
  engine_version        = var.instance_engine_version
  capacity              = var.instance_capacity
  flavor                = (var.instance_flavor_id != "" ? var.instance_flavor_id :
  try(data.huaweicloud_dcs_flavors.test[0].flavors[0].name, null))
  availability_zones    = (var.availability_zone != "" ? [var.availability_zone] :
  try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null))
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  password              = (var.instance_password != "" ? var.instance_password :
  try(random_password.test[0].result, null))
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew

  lifecycle {
    ignore_changes = [
      flavor,
      availability_zones,
    ]
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 instance_name 进行赋值
- **engine**：缓存引擎，固定为 Redis
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值
- **engine_version**：通过引用输入变量 instance_engine_version 进行赋值
- **capacity**：通过引用输入变量 instance_capacity 进行赋值
- **flavor**：当输入变量 instance_flavor_id 不为空时使用该值，否则使用查询到的产品规格名称
- **availability_zones**：当输入变量 availability_zone 不为空时使用该值，否则使用查询到的可用分区列表中的第一个
- **vpc_id**：通过引用资源 huaweicloud_vpc.test 的 ID 进行赋值
- **subnet_id**：通过引用资源 huaweicloud_vpc_subnet.test 的 ID 进行赋值
- **password**：当输入变量 instance_password 不为空时使用该值，否则使用随机生成的密码
- **charging_mode**、**period_unit**、**period**、**auto_renew**：通过引用对应输入变量进行赋值

### 8. 查询DCS实例分片列表

在TF文件（如main.tf）中添加以下脚本以查询DCS实例分片列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DCS实例分片列表
data "huaweicloud_dcs_instance_shards" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
}

locals {
  replication_list = try(data.huaweicloud_dcs_instance_shards.test.group_list[0].replication_list, [])
}
```

**参数说明**：
- **instance_id**：通过引用资源 huaweicloud_dcs_instance.test 的 ID 进行赋值
- **locals.replication_list**：从分片查询结果中提取副本列表，用于后续获取节点ID

### 9. 创建DCS会话清理

在TF文件（如main.tf）中添加以下脚本以执行会话清理：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DCS会话清理资源
variable "client_addrs" {
  description = "The list of client addresses to be killed"
  type        = list(string)
  default     = ["127.0.0.1:6379"]
}

resource "huaweicloud_dcs_sessions_kill" "test" {
  instance_id  = huaweicloud_dcs_instance.test.id
  node_id      = try(local.replication_list[0].node_id, "")
  client_addrs = var.client_addrs
}
```

**参数说明**：
- **instance_id**：通过引用资源 huaweicloud_dcs_instance.test 的 ID 进行赋值
- **node_id**：从分片查询结果中获取第一个副本的节点ID
- **client_addrs**：通过引用输入变量 client_addrs 进行赋值，指定需要清理的客户端地址列表

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
vpc_name      = "tf_test_dcs_instance_vpc"
subnet_name   = "tf_test_dcs_instance_subnet"
instance_name = "tf_test_dcs_instance"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DCS Redis实例并执行会话清理
4. 运行 `terraform show` 查看已创建的DCS Redis实例及会话清理结果

## 参考信息

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DCS Redis会话清理最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-sessions-kill)
