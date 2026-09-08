# 部署Redis大Key分析

## 应用场景

分布式缓存服务（DCS）的Redis实例在运行过程中，可能会产生大Key（即单个Key的Value过大或集合类Key的元素过多），这些大Key会导致实例性能下降、内存占用过高，甚至引发请求超时等问题。通过定期对大Key进行分析，可以及时发现并处理这些潜在风险，保障业务的稳定运行。

本最佳实践将介绍如何使用Terraform自动化部署DCS Redis实例并创建大Key分析任务，包括创建VPC、子网、Redis实例以及发起大Key分析。

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
- [DCS大Key分析（huaweicloud_dcs_bigkey_analysis）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_bigkey_analysis)

### 资源/数据源依赖关系

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
          └── huaweicloud_dcs_instance
                └── huaweicloud_dcs_bigkey_analysis

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

在TF文件（如main.tf）中添加以下脚本：

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
- **name**：通过引用输入变量 vpc_name 进行赋值，VPC的名称。
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值，VPC的地址范围。

### 3. 创建子网

在TF文件（如main.tf）中添加以下脚本：

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
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值，VPC的ID。
- **name**：通过引用输入变量 subnet_name 进行赋值，子网的名称。
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，子网的地址范围，若未指定则自动从VPC的CIDR中划分。
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，子网的网关地址，若未指定则自动计算。

### 4. 查询可用分区和DCS产品规格

在TF文件（如main.tf）中添加以下脚本：

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
- **huaweicloud_availability_zones.test**：当未指定可用分区时，查询可用的可用分区列表。
- **huaweicloud_dcs_flavors.test**：当未指定实例规格时，根据缓存类型、容量和引擎版本查询可用的DCS产品规格。

### 5. 生成随机密码

在TF文件（如main.tf）中添加以下脚本：

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

locals {
  dcs_password = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, "")
}
```

**参数说明**：
- **random_password.test**：当未指定实例密码时，生成一个随机密码，满足复杂度要求。
- **locals.dcs_password**：用于获取最终使用的密码，优先使用用户提供的密码，否则使用随机生成的密码。

### 6. 创建DCS Redis实例

在TF文件（如main.tf）中添加以下脚本：

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
  password           = local.dcs_password
  flavor             = (var.instance_flavor_id != "" ? var.instance_flavor_id :
  try(data.huaweicloud_dcs_flavors.test[0].flavors[0].name, null))
  availability_zones = (var.availability_zone != "" ? [var.availability_zone] :
  try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null))
}
```

**参数说明**：
- **name**：通过引用输入变量 instance_name 进行赋值，实例的名称。
- **engine**：缓存引擎，固定为Redis。
- **engine_version**：通过引用输入变量 instance_engine_version 进行赋值，引擎版本。
- **capacity**：通过引用输入变量 instance_capacity 进行赋值，实例容量（GB）。
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值，所属VPC的ID。
- **subnet_id**：通过引用 huaweicloud_vpc_subnet.test.id 进行赋值，所属子网的ID。
- **password**：通过引用 local.dcs_password 进行赋值，实例的访问密码。
- **flavor**：通过引用输入变量 instance_flavor_id 或 data.huaweicloud_dcs_flavors.test 进行赋值，实例规格。
- **availability_zones**：通过引用输入变量 availability_zone 或 data.huaweicloud_availability_zones.test 进行赋值，实例的可用分区。

### 7. 创建大Key分析任务

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建大Key分析任务
resource "huaweicloud_dcs_bigkey_analysis" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
}
```

**参数说明**：
- **instance_id**：通过引用 huaweicloud_dcs_instance.test.id 进行赋值，待分析的DCS Redis实例ID。

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
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

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DCS Redis实例和大Key分析任务
4. 运行 `terraform show` 查看已创建的DCS Redis实例和大Key分析任务

## 参考信息

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DCS Redis大Key分析最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-bigkey-analysis)
