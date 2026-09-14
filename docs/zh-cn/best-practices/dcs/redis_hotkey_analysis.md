# 部署Redis热Key分析

## 应用场景

分布式缓存服务（DCS）Redis实例在运行过程中，某些Key可能被高频访问，形成热Key。热Key会导致单个分片或节点负载过高，进而影响整个实例的响应性能，甚至引发业务抖动。通过热Key分析，可以识别出实例中被高频访问的Key，为业务优化、分片调整和缓存策略改进提供依据。

本最佳实践将介绍如何使用Terraform自动化部署一个DCS Redis实例，并在该实例上创建热Key分析任务，包括VPC创建、子网配置、可用分区与产品规格查询、实例创建以及热Key分析任务的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用分区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS产品规格列表（data.huaweicloud_dcs_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DCS实例（huaweicloud_dcs_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS热Key分析（huaweicloud_dcs_hotkey_analysis）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_hotkey_analysis)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
            └── huaweicloud_dcs_hotkey_analysis

random_password
    └── huaweicloud_dcs_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云和子网

在TF文件（如main.tf）中添加以下脚本以创建VPC和子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云和子网资源
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

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

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
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

- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，通过引用输入变量 vpc_cidr 进行赋值，默认值为`192.168.0.0/16`
- **vpc_id**：子网所属的VPC ID，通过引用 `huaweicloud_vpc.test.id` 进行赋值
- **cidr**（子网）：子网网段，当输入变量 subnet_cidr 为空时，通过 `cidrsubnet` 函数基于VPC网段自动计算
- **gateway_ip**：子网网关IP，当输入变量 subnet_gateway_ip 为空时，通过 `cidrhost` 函数基于子网网段自动计算

### 3. 查询可用分区和DCS产品规格

在TF文件（如main.tf）中添加以下脚本以查询可用分区和DCS产品规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用分区和DCS产品规格数据源
variable "availability_zone" {
  description = "The availability zone to which the Redis single instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_id" {
  description = "The flavor ID of the Redis single instance"
  type        = string
  default     = ""
}

variable "instance_capacity" {
  description = "The capacity of the Redis instance (in GB)"
  type        = number
  default     = 0.125
}

variable "instance_engine_version" {
  description = "The engine version of the Redis single instance"
  type        = string
  default     = "5.0"
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  cache_mode     = "ha"
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**参数说明**：

- **count**（可用分区）：当输入变量 availability_zone 为空时查询可用分区列表
- **cache_mode**：缓存模式，此处固定为`ha`（主备模式）
- **capacity**：实例容量，通过引用输入变量 instance_capacity 进行赋值，默认值为`0.125`GB
- **engine_version**：引擎版本，通过引用输入变量 instance_engine_version 进行赋值，默认值为`5.0`

### 4. 创建随机密码

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

- **count**：当输入变量 instance_password 为空时创建随机密码
- **length**：密码长度，此处为`12`
- **special**：是否包含特殊字符，此处为`true`
- **override_special**：允许使用的特殊字符集合
- **min_upper**、**min_lower**、**min_numeric**、**min_special**：分别指定大写字母、小写字母、数字和特殊字符的最小数量

### 5. 创建DCS Redis实例

在TF文件（如main.tf）中添加以下脚本以创建DCS Redis实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DCS Redis实例资源
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

  parameters {
    id    = "2"
    name  = "maxmemory-policy"
    value = "volatile-lfu"
  }
}
```

**参数说明**：

- **name**：实例名称，通过引用输入变量 instance_name 进行赋值
- **engine**：缓存引擎，此处固定为`Redis`
- **engine_version**：引擎版本，通过引用输入变量 instance_engine_version 进行赋值
- **capacity**：实例容量，通过引用输入变量 instance_capacity 进行赋值
- **vpc_id**：实例所属的VPC ID，通过引用 `huaweicloud_vpc.test.id` 进行赋值
- **subnet_id**：实例所属的子网ID，通过引用 `huaweicloud_vpc_subnet.test.id` 进行赋值
- **password**：实例密码，当输入变量 instance_password 不为空时使用该值，否则使用随机生成的密码
- **flavor**：实例规格，当输入变量 instance_flavor_id 不为空时使用该值，否则使用查询到的产品规格名称
- **availability_zones**：实例所属的可用分区，当输入变量 availability_zone 不为空时使用该值，否则使用查询到的可用分区列表中的第一个
- **parameters**：实例参数配置，此处将`maxmemory-policy`设置为`volatile-lfu`

### 6. 创建DCS热Key分析任务

在TF文件（如main.tf）中添加以下脚本以创建DCS热Key分析任务：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DCS热Key分析任务资源
resource "huaweicloud_dcs_hotkey_analysis" "test" {
  instance_id = huaweicloud_dcs_instance.test.id
}
```

**参数说明**：

- **instance_id**：需要进行热Key分析的DCS实例ID，通过引用 `huaweicloud_dcs_instance.test.id` 进行赋值

### 7. 预设资源部署所需的入参（可选）

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

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DCS Redis实例及热Key分析任务
4. 运行 `terraform show` 查看已创建的DCS Redis实例及热Key分析任务

## 参考信息

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DCS Redis热Key分析最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-hotkey-analysis)
