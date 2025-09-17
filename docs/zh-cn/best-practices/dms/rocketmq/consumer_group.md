# 部署RocketMQ消费者组

## 应用场景

华为云分布式消息服务RocketMQ版是一种高可用、高可靠、高性能的分布式消息中间件服务，广泛应用于电商、金融、IoT等行业的分布式系统中。
消费者组是RocketMQ消息消费的重要概念，用于管理消息的消费行为，确保消息能够被正确、高效地消费。

通过RocketMQ消费者组，企业可以实现消息的负载均衡消费、消费进度管理、消费失败重试等功能，满足不同业务场景下的消息消费需求。
本最佳实践将介绍如何使用Terraform自动化部署一个RocketMQ消费者组，包括RocketMQ实例和消费者组的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [RocketMQ可用区列表查询数据源（data.huaweicloud_dms_rocketmq_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_rocketmq_availability_zones)
- [RocketMQ规格列表查询数据源（data.huaweicloud_dms_rocketmq_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_rocketmq_flavors)
- [RocketMQ Broker查询数据源（data.huaweicloud_dms_rocketmq_broker）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_rocketmq_broker)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [RocketMQ实例资源（huaweicloud_dms_rocketmq_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rocketmq_instance)
- [RocketMQ消费者组资源（huaweicloud_dms_rocketmq_consumer_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rocketmq_consumer_group)

### 资源/数据源依赖关系

```text
data.huaweicloud_dms_rocketmq_availability_zones
    └── data.huaweicloud_dms_rocketmq_flavors
        └── huaweicloud_dms_rocketmq_instance
            └── data.huaweicloud_dms_rocketmq_broker
                └── huaweicloud_dms_rocketmq_consumer_group

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dms_rocketmq_instance
            └── huaweicloud_dms_rocketmq_consumer_group

huaweicloud_networking_secgroup
    └── huaweicloud_dms_rocketmq_instance
        └── huaweicloud_dms_rocketmq_consumer_group
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询RocketMQ可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建RocketMQ实例：

```hcl
variable "availability_zones" {
  description = "The availability zones to which the RocketMQ instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的RocketMQ可用区信息，用于创建RocketMQ实例
data "huaweicloud_dms_rocketmq_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**参数说明**：

- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当`var.availability_zones`为空时创建数据源（即执行可用区列表查询）

### 3. 通过数据源查询RocketMQ规格信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建RocketMQ实例：

```hcl
variable "instance_flavor_id" {
  description = "The flavor ID of the RocketMQ instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_type" {
  description = "The flavor type of the RocketMQ instance"
  type        = string
  default     = "cluster.small"
}

variable "availability_zones_count" {
  description = "The number of availability zones"
  type        = number
  default     = 1
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的RocketMQ规格信息，用于创建RocketMQ实例
data "huaweicloud_dms_rocketmq_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  type               = var.instance_flavor_type
  availability_zones = length(var.availability_zones) != 0 ? var.availability_zones : slice(data.huaweicloud_dms_rocketmq_availability_zones.test[0].availability_zones[*].code, 0, var.availability_zones_count)
}
```

**参数说明**：

- **count**：数据源的创建数，用于控制是否执行规格列表查询数据源，仅当 `var.instance_flavor_id` 为空时创建数据源（即执行规格列表查询）
- **type**：RocketMQ实例规格类型，优先使用输入变量中指定的规格类型，如未指定则默认为"cluster.small"
- **availability_zones**：可用区列表，优先使用输入变量中指定的可用区，如未指定则使用数据源查询结果中的前 `var.availability_zones_count` 个可用区
- **availability_zones_count**：可用区数量，优先使用输入变量中指定的可用区数量，如未指定则默认为1

### 4. 创建VPC资源

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

- **name**：VPC的名称
- **cidr**：VPC的CIDR块，优先使用输入变量中指定的CIDR块，如未指定则默认为"192.168.0.0/16"

### 5. 创建VPC子网资源

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
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：

- **vpc_id**：子网所属VPC的ID，引用前面创建的VPC资源的ID
- **name**：子网的名称
- **cidr**：子网的CIDR块，优先使用输入变量中指定的CIDR块，如未指定则自动计算
- **gateway_ip**：子网的网关IP，优先使用输入变量中指定的网关IP，如未指定则自动计算

### 6. 创建安全组资源

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

- **name**：安全组的名称
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 7. 创建RocketMQ实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建RocketMQ实例资源：

```hcl
variable "instance_name" {
  description = "The name of the RocketMQ instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the RocketMQ instance"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.instance_flavor_id == "" || var.instance_engine_version != ""
    error_message = "When 'instance_flavor_id' is not empty, 'instance_engine_version' is required"
  }
}

variable "instance_storage_spec_code" {
  description = "The storage spec code of the RocketMQ instance"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.instance_flavor_id == "" || var.instance_storage_spec_code != ""
    error_message = "When 'instance_flavor_id' is not empty, 'instance_storage_spec_code' is required"
  }
}

variable "instance_storage_space" {
  description = "The storage space of the RocketMQ instance in GB"
  type        = number
  default     = 800
}

variable "instance_broker_num" {
  description = "The number of brokers for the RocketMQ instance"
  type        = number
  default     = 1
}

variable "instance_description" {
  description = "The description of the RocketMQ instance"
  type        = string
  default     = ""
}

variable "instance_tags" {
  description = "The tags of the RocketMQ instance"
  type        = map(string)
  default     = {}
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = null
}

variable "instance_enable_acl" {
  description = "Whether to enable ACL for the RocketMQ instance"
  type        = bool
  default     = false
}

variable "instance_tls_mode" {
  description = "The TLS mode of the RocketMQ instance"
  type        = string
  default     = "SSL"
}

variable "instance_configs" {
  description = "The configuration parameters of the RocketMQ instance"
  type = list(object({
    name  = string
    value = string
  }))

  nullable    = false
  default = []
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RocketMQ实例资源
resource "huaweicloud_dms_rocketmq_instance" "test" {
  name                  = var.instance_name
  flavor_id             = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_dms_rocketmq_flavors.test[0].flavors[0].id, null)
  engine_version        = var.instance_engine_version != "" ? var.instance_engine_version : try(data.huaweicloud_dms_rocketmq_flavors.test[0].versions[0], null)
  storage_spec_code     = var.instance_storage_spec_code != "" ? var.instance_storage_spec_code : try(data.huaweicloud_dms_rocketmq_flavors.test[0].flavors[0].ios[0].storage_spec_code, null)
  storage_space         = var.instance_storage_space
  availability_zones    = length(var.availability_zones) != 0 ? var.availability_zones : slice(data.huaweicloud_dms_rocketmq_availability_zones.test[0].availability_zones[*].code, 0, var.availability_zones_count)
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  broker_num            = var.instance_broker_num
  description           = var.instance_description
  tags                  = var.instance_tags
  enterprise_project_id = var.enterprise_project_id
  enable_acl            = var.instance_enable_acl
  tls_mode              = var.instance_tls_mode

  dynamic "configs" {
    for_each = var.instance_configs

    content {
      name  = configs.value.name
      value = configs.value.value
    }
  }
}
```

**参数说明**：

- **name**：RocketMQ实例的名称
- **flavor_id**：RocketMQ实例规格ID，优先使用输入变量中指定的规格，如未指定则使用规格查询数据源的结果
- **engine_version**：RocketMQ引擎版本，优先使用输入变量中指定的版本，如未指定则使用规格查询数据源的结果
- **storage_spec_code**：存储规格代码，优先使用输入变量中指定的规格，如未指定则使用规格查询数据源的结果
- **storage_space**：存储空间大小，优先使用输入变量中指定的存储空间大小，如未指定则默认为800GB
- **availability_zones**：可用区列表，优先使用输入变量中指定的可用区，如未指定则使用可用区查询数据源的结果
- **vpc_id**：VPC ID，引用前面创建的VPC资源的ID
- **subnet_id**：子网ID，引用前面创建的子网资源的ID
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **broker_num**：Broker数量，优先使用输入变量中指定的Broker数量，如未指定则默认为1
- **description**：实例的描述
- **tags**：实例标签
- **enterprise_project_id**：企业项目ID
- **enable_acl**：是否启用ACL，优先使用输入变量中指定的是否启用ACL，如未指定则默认为false
- **tls_mode**：TLS模式，优先使用输入变量中指定的TLS模式，如未指定则默认为"SSL"
- **configs**：实例配置块

### 8. 通过数据源查询RocketMQ Broker信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建消费者组：

```hcl
variable "consumer_group_brokers" {
  description = "The broker list of the consumer group, it's only valid when the RocketMQ instance version is `4.8.0`"
  type        = list(string)
  default     = []
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的RocketMQ Broker信息，用于创建消费者组
data "huaweicloud_dms_rocketmq_broker" "test" {
  count = length(var.consumer_group_brokers) == 0 && huaweicloud_dms_rocketmq_instance.test.engine_version == "4.8.0" ? 1 : 0

  instance_id = huaweicloud_dms_rocketmq_instance.test.id
}
```

**参数说明**：

- **count**：数据源的创建数，用于控制是否执行Broker查询数据源，仅当未指定消费者组Broker列表且RocketMQ实例版本为4.8.0时创建数据源
- **instance_id**：RocketMQ实例ID，引用前面创建的RocketMQ实例资源的ID

### 9. 创建RocketMQ消费者组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建RocketMQ消费者组资源：

```hcl
variable "consumer_group_name" {
  description = "The name of the RocketMQ consumer group"
  type        = string
}

variable "consumer_group_retry_max_times" {
  description = "The maximum retry times for failed consumption"
  type        = number
  default     = 16
}

variable "consumer_group_enabled" {
  description = "Whether to enable the consumer group"
  type        = bool
  default     = true
}

variable "consumer_group_broadcast" {
  description = "Whether to enable broadcast mode for the consumer group"
  type        = bool
  default     = false
}

variable "consumer_group_description" {
  description = "The description of the consumer group"
  type        = string
  default     = ""
}

variable "consumer_group_consume_orderly" {
  description = "Whether to enable orderly consumption for the consumer group"
  type        = bool
  default     = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RocketMQ消费者组资源
resource "huaweicloud_dms_rocketmq_consumer_group" "test" {
  instance_id     = huaweicloud_dms_rocketmq_instance.test.id
  name            = var.consumer_group_name
  retry_max_times = var.consumer_group_retry_max_times
  enabled         = var.consumer_group_enabled
  broadcast       = var.consumer_group_broadcast
  description     = var.consumer_group_description
  brokers         = length(var.consumer_group_brokers) > 0 ? var.consumer_group_brokers : try(data.huaweicloud_dms_rocketmq_broker.test[0].brokers, [])
  consume_orderly = var.consumer_group_consume_orderly
}
```

**参数说明**：

- **instance_id**：RocketMQ实例ID，引用前面创建的RocketMQ实例资源的ID
- **name**：消费者组名称
- **retry_max_times**：消费失败重试最大次数，优先使用输入变量中指定的消费失败重试最大次数，如未指定则默认为16次
- **enabled**：是否启用消费者组，优先使用输入变量中指定的是否启用消费者组，如未指定则默认为true
- **broadcast**：是否启用广播模式，优先使用输入变量中指定的是否启用广播模式，如未指定则默认为false
- **description**：消费者组的描述
- **brokers**：Broker列表，优先使用输入变量中指定的Broker列表，如未指定则使用Broker查询数据源的结果
- **consume_orderly**：是否启用顺序消费，优先使用输入变量中指定的是否启用顺序消费，如未指定则默认为false，仅在RocketMQ实例版本为5.x时有效

### 10. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC和子网配置
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"

# RocketMQ实例基本信息
instance_name       = "tf_test_instance"
instance_broker_num = 1

# 消费者组基本信息
consumer_group_name = "tf_test_consumer_group"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，
   其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建RocketMQ消费者组
4. 运行 `terraform show` 查看已创建的RocketMQ消费者组

## 参考信息

- [华为云分布式消息服务RocketMQ产品文档](https://support.huaweicloud.com/hrm/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RocketMQ消费者组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/rocketmq/consumer-group)
