# 部署Kafka实例配置

## 应用场景

华为云分布式消息服务Kafka版是一种高可用、高可靠、高性能的分布式消息中间件服务，广泛应用于大数据、日志收集、流式处理等场景。通过配置Kafka实例，您可以创建和管理Kafka集群，包括实例规格、存储配置、网络配置、安全配置等，实现消息的可靠传输和处理。通过Terraform自动化配置Kafka实例，可以确保实例配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化配置Kafka实例。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区数据源（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Kafka规格数据源（huaweicloud_dms_kafka_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Kafka实例资源（huaweicloud_dms_kafka_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)

### 资源/数据源依赖关系

```text
data.huaweicloud_availability_zones
    └── data.huaweicloud_dms_kafka_flavors
        └── huaweicloud_dms_kafka_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dms_kafka_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dms_kafka_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询数据源

在TF文件（如main.tf）中添加以下脚本以查询可用区和Kafka规格信息：

```hcl
# 查询可用区信息
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}

# 查询Kafka规格信息
data "huaweicloud_dms_kafka_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  type               = var.instance_flavor_type
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3)) : var.availability_zones
  storage_spec_code  = var.instance_storage_spec_code
}
```

**参数说明**：
- **type**：规格类型，通过引用输入变量instance_flavor_type进行赋值，默认值为"cluster"（集群模式）
- **availability_zones**：可用区列表，通过引用输入变量availability_zones或可用区数据源进行赋值
- **storage_spec_code**：存储规格代码，通过引用输入变量instance_storage_spec_code进行赋值，默认值为"dms.physical.storage.ultra.v2"

### 3. 创建基础网络资源

在TF文件（如main.tf）中添加以下脚本以创建VPC、子网和安全组：

```hcl
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
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
}

variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 创建VPC
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

# 创建子网
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}

# 创建安全组
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

### 4. 创建Kafka实例资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建Kafka实例资源：

```hcl
variable "availability_zones" {
  description = "The availability zones to which the Kafka instance belongs"
  type        = list(string)
  default     = []
}

variable "instance_name" {
  description = "The name of the Kafka instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the Kafka instance"
  type        = string
  default     = "2.7"
}

variable "instance_flavor_id" {
  description = "The flavor ID of the Kafka instance"
  type        = string
  default     = ""
}

variable "instance_flavor_type" {
  description = "The flavor type of the Kafka instance"
  type        = string
  default     = "cluster"
}

variable "instance_storage_spec_code" {
  description = "The storage specification code of the Kafka instance"
  type        = string
  default     = "dms.physical.storage.ultra.v2"
}

variable "instance_storage_space" {
  description = "The storage space of the Kafka instance"
  type        = number
  default     = 600
}

variable "instance_broker_num" {
  description = "The number of brokers of the Kafka instance"
  type        = number
  default     = 3
}

variable "instance_ssl_enable" {
  description = "Whether to enable SSL for the Kafka instance"
  type        = bool
  default     = false
}

variable "instance_access_user_name" {
  description = "The access user name of the Kafka instance"
  type        = string
  default     = ""
}

variable "instance_access_user_password" {
  description = "The access password of the Kafka instance"
  type        = string
  sensitive   = true
  default     = ""
}

variable "instance_description" {
  description = "The description of the Kafka instance"
  type        = string
  default     = ""
}

variable "charging_mode" {
  description = "The charging mode of the Kafka instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the Kafka instance"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the Kafka instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "Whether to enable auto renew for the Kafka instance"
  type        = string
  default     = "false"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Kafka实例资源
resource "huaweicloud_dms_kafka_instance" "test" {
  name               = var.instance_name
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3)) : var.availability_zones
  engine_version     = var.instance_engine_version
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_dms_kafka_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  storage_spec_code  = var.instance_storage_spec_code
  storage_space      = var.instance_storage_space
  broker_num         = var.instance_broker_num
  vpc_id             = huaweicloud_vpc.test.id
  network_id         = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  ssl_enable         = var.instance_ssl_enable
  access_user        = var.instance_access_user_name
  password           = var.instance_access_user_password
  description        = var.instance_description
  charging_mode      = var.charging_mode
  period_unit        = var.period_unit
  period             = var.period
  auto_renew         = var.auto_renew

  lifecycle {
    ignore_changes = [
      availability_zones,
      flavor_id,
    ]
  }
}
```

**参数说明**：
- **name**：Kafka实例名称，通过引用输入变量instance_name进行赋值
- **availability_zones**：可用区列表，通过引用输入变量availability_zones或可用区数据源进行赋值
- **engine_version**：引擎版本，通过引用输入变量instance_engine_version进行赋值，默认值为"2.7"
- **flavor_id**：规格ID，通过引用输入变量instance_flavor_id或Kafka规格数据源进行赋值
- **storage_spec_code**：存储规格代码，通过引用输入变量instance_storage_spec_code进行赋值，默认值为"dms.physical.storage.ultra.v2"
- **storage_space**：存储空间，通过引用输入变量instance_storage_space进行赋值，默认值为600（GB）
- **broker_num**：Broker数量，通过引用输入变量instance_broker_num进行赋值，默认值为3
- **vpc_id**：VPC ID，通过引用VPC资源进行赋值
- **network_id**：网络子网ID，通过引用子网资源进行赋值
- **security_group_id**：安全组ID，通过引用安全组资源进行赋值
- **ssl_enable**：是否启用SSL，通过引用输入变量instance_ssl_enable进行赋值，默认值为false
- **access_user**：访问用户名，通过引用输入变量instance_access_user_name进行赋值，可选参数，默认值为空字符串
- **password**：访问密码，通过引用输入变量instance_access_user_password进行赋值，可选参数，默认值为空字符串
- **description**：实例描述，通过引用输入变量instance_description进行赋值，可选参数，默认值为空字符串
- **charging_mode**：计费模式，通过引用输入变量charging_mode进行赋值，默认值为"postPaid"（按需计费）
- **period_unit**：计费周期单位，通过引用输入变量period_unit进行赋值，可选参数，默认值为null
- **period**：计费周期，通过引用输入变量period进行赋值，可选参数，默认值为null
- **auto_renew**：是否自动续费，通过引用输入变量auto_renew进行赋值，默认值为"false"

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC和子网配置
vpc_name    = "tf_test_instance"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_instance"

# 安全组配置
security_group_name = "tf_test_instance"

# Kafka实例配置
instance_name                 = "tf_test_instance"
instance_ssl_enable           = true
instance_access_user_name     = "admin"
instance_access_user_password = "YourKafkaInstancePassword!"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是`instance_access_user_password`需要设置符合密码复杂度要求的密码
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="instance_name=my_kafka" -var="vpc_name=my_vpc"`
2. 环境变量：`export TF_VAR_instance_name=my_kafka` 和 `export TF_VAR_vpc_name=my_vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。由于instance_access_user_password包含敏感信息，建议使用环境变量或加密的变量文件进行设置。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建Kafka实例：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Kafka实例及相关资源
4. 运行 `terraform show` 查看已创建的Kafka实例详情

> 注意：Kafka实例创建后，可以通过配置SSL和访问用户来增强安全性。实例的可用区和规格ID在创建后不能修改，因此需要在创建时正确配置。通过lifecycle.ignore_changes可以避免Terraform在后续更新时修改这些不可变参数。

## 参考信息

- [华为云分布式消息服务Kafka产品文档](https://support.huaweicloud.com/kafka/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [实例配置最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/instance-configuration)
