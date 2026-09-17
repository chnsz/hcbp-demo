# 部署RabbitMQ基础实例

## 应用场景

分布式消息服务（DMS）RabbitMQ是一款基于AMQP协议的高性能、高可靠消息中间件服务，广泛应用于异步通信、系统解耦、流量削峰填谷等业务场景。通过RabbitMQ实例，企业可以构建松耦合、可扩展的分布式应用架构，保障消息的可靠传输与高效流转。

本最佳实践将介绍如何使用Terraform自动化部署一个DMS RabbitMQ基础实例，包括VPC、子网、安全组以及RabbitMQ实例的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RabbitMQ实例规格（data.huaweicloud_dms_rabbitmq_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_rabbitmq_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [RabbitMQ实例（huaweicloud_dms_rabbitmq_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dms_rabbitmq_instance

data.huaweicloud_dms_rabbitmq_flavors
    └── huaweicloud_dms_rabbitmq_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dms_rabbitmq_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dms_rabbitmq_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询可用区列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
variable "availability_zones" {
  description = "The availability zones to which the RabbitMQ instance belongs"
  type        = list(string)
  default     = []
}

data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**参数说明**：
- **count**：当输入变量 availability_zones 为空时创建该数据源，用于自动获取当前region下的可用区列表

### 3. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云
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

### 4. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "192.168.0.0/24"
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = "192.168.0.1"
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，若为空则基于VPC的CIDR自动计算
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，若为空则基于VPC的CIDR自动计算

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
- **delete_default_rules**：设置为true以删除安全组的默认规则

### 6. 查询RabbitMQ实例规格

在TF文件（如main.tf）中添加以下脚本以查询RabbitMQ实例规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询RabbitMQ实例规格
variable "instance_flavor_id" {
  description = "The flavor ID of the RabbitMQ instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_type" {
  description = "The flavor type of the RabbitMQ instance"
  type        = string
  default     = "cluster"
}

variable "instance_storage_spec_code" {
  description = "The storage specification code of the RabbitMQ instance"
  type        = string
  default     = "dms.physical.storage.ultra.v2"
}

variable "availability_zone_number" {
  description = "The number of availability zones to which the RabbitMQ instance belongs"
  type        = number
  default     = 1
}

data "huaweicloud_dms_rabbitmq_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  type               = var.instance_flavor_type
  flavor_id          = var.instance_flavor_id
  storage_spec_code  = var.instance_storage_spec_code
  availability_zones = length(var.availability_zones) != 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, var.availability_zone_number), null)
}
```

**参数说明**：
- **count**：当输入变量 instance_flavor_id 为空时创建该数据源，用于自动获取可用的RabbitMQ实例规格
- **type**：通过引用输入变量 instance_flavor_type 进行赋值，可选值为 single 或 cluster
- **flavor_id**：通过引用输入变量 instance_flavor_id 进行赋值
- **storage_spec_code**：通过引用输入变量 instance_storage_spec_code 进行赋值
- **availability_zones**：通过引用输入变量 availability_zones 进行赋值，若为空则基于可用区列表和输入变量 availability_zone_number 自动截取

### 7. 创建RabbitMQ实例

在TF文件（如main.tf）中添加以下脚本以创建RabbitMQ实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RabbitMQ实例
variable "instance_name" {
  description = "The name of the RabbitMQ instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the RabbitMQ instance"
  type        = string
  default     = "3.8.35"
}

variable "instance_broker_num" {
  description = "The number of brokers of the RabbitMQ instance"
  type        = number
  default     = 3
}

variable "instance_storage_space" {
  description = "The storage space of the RabbitMQ instance"
  type        = number
  default     = 600
}

variable "instance_ssl_enable" {
  description = "The SSL enable of the RabbitMQ instance"
  type        = bool
  default     = false
}

variable "instance_access_user_name" {
  description = "The access user of the RabbitMQ instance"
  type        = string
}

variable "instance_password" {
  description = "The access password of the RabbitMQ instance"
  sensitive   = true
  type        = string
}

variable "instance_description" {
  description = "The description of the RabbitMQ instance"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the RabbitMQ instance belongs"
  type        = string
  default     = null
}

variable "instance_tags" {
  description = "The key/value pairs to associate with the instance"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the RabbitMQ instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the RabbitMQ instance"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the RabbitMQ instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the RabbitMQ instance"
  type        = string
  default     = "false"
}

resource "huaweicloud_dms_rabbitmq_instance" "test" {
  name                  = var.instance_name
  engine_version        = var.instance_engine_version
  flavor_id             = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_dms_rabbitmq_flavors.test[0].flavors[0].id, null)
  vpc_id                = huaweicloud_vpc.test.id
  network_id            = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zones    = length(var.availability_zones) != 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, var.availability_zone_number), null)
  broker_num            = var.instance_broker_num
  storage_space         = var.instance_storage_space
  storage_spec_code     = var.instance_storage_spec_code
  ssl_enable            = var.instance_ssl_enable
  access_user           = var.instance_access_user_name
  password              = var.instance_password
  description           = var.instance_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.instance_tags
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew

  # If you want to change some of the following parameters, you need to remove the corresponding fields from "lifecycle.ignore_changes".
  lifecycle {
    ignore_changes = [
      flavor_id,
      availability_zones,
    ]
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 instance_name 进行赋值
- **engine_version**：通过引用输入变量 instance_engine_version 进行赋值
- **flavor_id**：通过引用输入变量 instance_flavor_id 进行赋值，若为空则使用数据源查询到的第一个规格ID
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **network_id**：通过引用虚拟私有云子网资源的ID进行赋值
- **security_group_id**：通过引用安全组资源的ID进行赋值
- **availability_zones**：通过引用输入变量 availability_zones 进行赋值，若为空则基于可用区列表和输入变量 availability_zone_number 自动截取
- **broker_num**：通过引用输入变量 instance_broker_num 进行赋值，单机实例该值只能为1
- **storage_space**：通过引用输入变量 instance_storage_space 进行赋值
- **storage_spec_code**：通过引用输入变量 instance_storage_spec_code 进行赋值
- **ssl_enable**：通过引用输入变量 instance_ssl_enable 进行赋值
- **access_user**：通过引用输入变量 instance_access_user_name 进行赋值
- **password**：通过引用输入变量 instance_password 进行赋值
- **description**：通过引用输入变量 instance_description 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值
- **tags**：通过引用输入变量 instance_tags 进行赋值
- **charging_mode**：通过引用输入变量 charging_mode 进行赋值
- **period_unit**：通过引用输入变量 period_unit 进行赋值
- **period**：通过引用输入变量 period 进行赋值
- **auto_renew**：通过引用输入变量 auto_renew 进行赋值

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name                  = "tf_test_rabbitmq_vpc"
subnet_name               = "tf_test_rabbitmq_subnet"
security_group_name       = "tf_test_rabbitmq_security_group"
instance_name             = "tf_test_rabbitmq"
instance_access_user_name = "tf_test_rabbitmq_user"
instance_password         = "YourPassword!"
instance_tags = {
  owner = "terraform"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建RabbitMQ实例
4. 运行 `terraform show` 查看已创建的RabbitMQ实例

## 参考信息

- [华为云分布式消息服务RabbitMQ产品文档](https://support.huaweicloud.com/rabbitmq/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DMS RabbitMQ基础实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/rabbitmq/basic-instance)
