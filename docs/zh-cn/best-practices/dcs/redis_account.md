# 部署Redis账号管理

## 应用场景

分布式缓存服务（Distributed Cache Service, DCS）是华为云提供的高性能、高可用的内存数据库服务，支持Redis、Memcached等主流缓存引擎。在实际业务中，为了满足多用户、多应用的隔离访问需求，通常需要在DCS Redis实例上创建独立的账号，并为其分配只读或读写权限，以实现细粒度的访问控制。

本最佳实践将介绍如何使用Terraform完成DCS Redis实例及其账号的自动化部署，包括创建VPC、子网、Redis实例以及DCS账号，并配置相应的访问权限。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用分区（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS产品规格（huaweicloud_dcs_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### 资源

- [虚拟私有云VPC（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DCS Redis实例（huaweicloud_dcs_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS账号（huaweicloud_dcs_account）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_account)

### 资源/数据源依赖关系

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
          └── huaweicloud_dcs_instance
                └── huaweicloud_dcs_account

data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

random_password
    └── huaweicloud_dcs_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC

在TF文件（如main.tf）中添加以下脚本，创建VPC：

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

- **name**：通过引用输入变量vpc_name进行赋值。
- **cidr**：通过引用输入变量vpc_cidr进行赋值。

### 3. 创建子网

在TF文件（如main.tf）中添加以下脚本，创建子网：

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

- **vpc_id**：通过引用已创建的huaweicloud_vpc.test的ID进行赋值。
- **name**：通过引用输入变量subnet_name进行赋值。
- **cidr**：通过引用输入变量subnet_cidr进行赋值，若未指定则自动从VPC的CIDR中划分。
- **gateway_ip**：通过引用输入变量subnet_gateway_ip进行赋值，若未指定则自动计算。

### 4. 查询可用分区和DCS产品规格

在TF文件（如main.tf）中添加以下脚本，查询可用分区和DCS产品规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用分区
variable "availability_zone" {
  description = "The availability zone to which the Redis single instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DCS产品规格
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
  default     = "5.0"
}

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  cache_mode     = "ha"
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**参数说明**：

- **availability_zone**：通过引用输入变量availability_zone进行赋值，若未指定则查询可用分区。
- **instance_flavor_id**：通过引用输入变量instance_flavor_id进行赋值，若未指定则查询产品规格。
- **cache_mode**：固定为"ha"，表示主备模式。
- **capacity**：通过引用输入变量instance_capacity进行赋值。
- **engine_version**：通过引用输入变量instance_engine_version进行赋值。

### 5. 生成随机密码

在TF文件（如main.tf）中添加以下脚本，当未指定实例密码时生成随机密码：

```hcl
# 生成随机密码
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

- **length**：密码长度，固定为12。
- **special**：是否包含特殊字符，固定为true。
- **override_special**：指定可用的特殊字符集合。
- **min_upper**：至少包含的大写字母数。
- **min_lower**：至少包含的小写字母数。
- **min_numeric**：至少包含的数字数。
- **min_special**：至少包含的特殊字符数。

### 6. 创建DCS Redis实例

在TF文件（如main.tf）中添加以下脚本，创建DCS Redis实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DCS Redis实例
variable "instance_name" {
  description = "The name of the Redis single instance"
  type        = string
}

resource "huaweicloud_dcs_instance" "test" {
  name               = var.instance_name
  engine             = "Redis"
  engine_version     = var.instance_engine_version
  capacity           = var.instance_capacity
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  password           = (var.instance_password != "" ? var.instance_password :
  try(random_password.test[0].result, null))
  flavor             = (var.instance_flavor_id != "" ? var.instance_flavor_id :
  try(data.huaweicloud_dcs_flavors.test[0].flavors[0].name, null))
  availability_zones = (var.availability_zone != "" ? [var.availability_zone] :
  try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null))
}
```

**参数说明**：

- **name**：通过引用输入变量instance_name进行赋值。
- **engine**：固定为"Redis"。
- **engine_version**：通过引用输入变量instance_engine_version进行赋值。
- **capacity**：通过引用输入变量instance_capacity进行赋值。
- **vpc_id**：通过引用已创建的huaweicloud_vpc.test的ID进行赋值。
- **subnet_id**：通过引用已创建的huaweicloud_vpc_subnet.test的ID进行赋值。
- **password**：通过引用输入变量instance_password进行赋值，若未指定则使用random_password生成的密码。
- **flavor**：通过引用输入变量instance_flavor_id进行赋值，若未指定则使用查询到的产品规格。
- **availability_zones**：通过引用输入变量availability_zone进行赋值，若未指定则使用查询到的可用分区。

### 7. 创建DCS账号

在TF文件（如main.tf）中添加以下脚本，创建DCS账号：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DCS账号
variable "account_name" {
  description = "The name of the DCS account"
  type        = string
}

variable "account_role" {
  description = "The role of the DCS account. Valid values: read, write"
  type        = string
  default     = "read"
}

variable "account_password" {
  description = "The password of the DCS account"
  type        = string
  sensitive   = true
}

variable "account_description" {
  description = "The description of the DCS account"
  type        = string
  default     = ""
}

resource "huaweicloud_dcs_account" "test" {
  instance_id      = huaweicloud_dcs_instance.test.id
  account_name     = var.account_name
  account_role     = var.account_role
  account_password = var.account_password
  description      = var.account_description
}
```

**参数说明**：

- **instance_id**：通过引用已创建的huaweicloud_dcs_instance.test的ID进行赋值。
- **account_name**：通过引用输入变量account_name进行赋值。
- **account_role**：通过引用输入变量account_role进行赋值，可选值为read和write。
- **account_password**：通过引用输入变量account_password进行赋值。
- **description**：通过引用输入变量account_description进行赋值。

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
vpc_name         = "tf_test_dcs_instance_vpc2"
subnet_name      = "tf_test_dcs_instance_subnet"
instance_name    = "tf_test_dcs_instance"
account_name     = "tf_test_account"
account_password = "Terraform@123"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DCS Redis实例及账号
4. 运行 `terraform show` 查看已创建的DCS Redis实例及账号

## 参考信息

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DCS Redis账号管理最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-account)
