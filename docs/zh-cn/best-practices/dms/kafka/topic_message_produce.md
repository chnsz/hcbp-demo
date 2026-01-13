# 部署主题消息生产

## 应用场景

华为云分布式消息服务Kafka版是一种高可用、高可靠、高性能的分布式消息中间件服务，广泛应用于大数据、日志收集、流式处理等场景。通过Kafka主题消息生产功能，您可以将消息发送到指定的Kafka主题，实现消息的可靠传输和处理。通过Terraform自动化部署Kafka主题消息生产，可以确保消息生产配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化部署Kafka主题消息生产，包括创建Kafka实例、主题和消息生产。

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
- [Kafka主题资源（huaweicloud_dms_kafka_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_topic)
- [Kafka消息生产资源（huaweicloud_dms_kafka_message_produce）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_message_produce)

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

huaweicloud_dms_kafka_instance
    └── huaweicloud_dms_kafka_topic
        └── huaweicloud_dms_kafka_message_produce
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

# 查询Kafka规格信息（仅在未指定flavor_id时查询）
data "huaweicloud_dms_kafka_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  type               = var.instance_flavor_type
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  storage_spec_code  = var.instance_storage_spec_code
}
```

**参数说明**：
- **type**：规格类型，通过引用输入变量instance_flavor_type进行赋值，默认值为"cluster"（集群模式）
- **availability_zones**：可用区列表，通过引用输入变量或可用区数据源进行赋值
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
  name = var.security_group_name
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

variable "instance_name" {
  description = "The name of the Kafka instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the Kafka instance"
  type        = string
  default     = "2.7"
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

variable "instance_access_user_name" {
  description = "The access user of the Kafka instance"
  type        = string
  default     = ""
}

variable "instance_access_user_password" {
  description = "The access password of the Kafka instance"
  type        = string
  default     = ""
  sensitive   = true
}

variable "instance_enabled_mechanisms" {
  description = "The enabled mechanisms of the Kafka instance"
  type        = list(string)
  default     = ["PLAIN"]
}

variable "port_protocol" {
  description = "The port protocol of the Kafka instance"

  type = object({
    private_plain_enable          = optional(bool, null)
    private_sasl_ssl_enable       = optional(bool, null)
    private_sasl_plaintext_enable = optional(bool, null)
    public_plain_enable           = optional(bool, null)
    public_sasl_ssl_enable        = optional(bool, null)
    public_sasl_plaintext_enable  = optional(bool, null)
  })

  default = {
    private_plain_enable = true
  }
}

# 创建Kafka实例
resource "huaweicloud_dms_kafka_instance" "test" {
  name               = var.instance_name
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  engine_version     = var.instance_engine_version
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_dms_kafka_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  storage_spec_code  = var.instance_storage_spec_code
  storage_space      = var.instance_storage_space
  broker_num         = var.instance_broker_num
  vpc_id             = huaweicloud_vpc.test.id
  network_id         = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  access_user        = var.instance_access_user_name
  password           = var.instance_access_user_password
  enabled_mechanisms = var.instance_enabled_mechanisms

  dynamic "port_protocol" {
    for_each = [var.port_protocol]

    content {
      private_plain_enable          = port_protocol.value.private_plain_enable
      private_sasl_ssl_enable       = port_protocol.value.private_sasl_ssl_enable
      private_sasl_plaintext_enable = port_protocol.value.private_sasl_plaintext_enable
      public_plain_enable           = port_protocol.value.public_plain_enable
      public_sasl_ssl_enable        = port_protocol.value.public_sasl_ssl_enable
      public_sasl_plaintext_enable  = port_protocol.value.public_sasl_plaintext_enable
    }
  }

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
- **availability_zones**：可用区列表，通过引用输入变量或可用区数据源进行赋值
- **engine_version**：引擎版本，通过引用输入变量instance_engine_version进行赋值，默认值为"2.7"
- **flavor_id**：规格ID，通过引用输入变量或Kafka规格数据源进行赋值
- **storage_spec_code**：存储规格代码，通过引用输入变量instance_storage_spec_code进行赋值，默认值为"dms.physical.storage.ultra.v2"
- **storage_space**：存储空间，通过引用输入变量instance_storage_space进行赋值，默认值为600（GB）
- **broker_num**：Broker数量，通过引用输入变量instance_broker_num进行赋值，默认值为3
- **vpc_id**：VPC ID，通过引用VPC资源进行赋值
- **network_id**：网络子网ID，通过引用子网资源进行赋值
- **security_group_id**：安全组ID，通过引用安全组资源进行赋值
- **access_user**：访问用户名，通过引用输入变量instance_access_user_name进行赋值，可选参数
- **password**：访问密码，通过引用输入变量instance_access_user_password进行赋值，可选参数
- **enabled_mechanisms**：启用的认证机制，通过引用输入变量instance_enabled_mechanisms进行赋值，默认值为["PLAIN"]
- **port_protocol**：端口协议配置，通过动态块进行配置，支持私有和公网访问的多种协议

### 5. 创建Kafka主题资源

在TF文件（如main.tf）中添加以下脚本以创建Kafka主题：

```hcl
variable "topic_name" {
  description = "The name of the topic"
  type        = string
}

variable "topic_partitions" {
  description = "The number of partitions of the topic"
  type        = number
  default     = 10
}

variable "topic_replicas" {
  description = "The number of replicas of the topic"
  type        = number
  default     = 3
}

variable "topic_aging_time" {
  description = "The aging time of the topic"
  type        = number
  default     = 72
}

variable "topic_sync_replication" {
  description = "The sync replication of the topic"
  type        = bool
  default     = false
}

variable "topic_sync_flushing" {
  description = "The sync flushing of the topic"
  type        = bool
  default     = false
}

variable "topic_description" {
  description = "The description of the topic"
  type        = string
  default     = null
}

variable "topic_configs" {
  description = "The configs of the topic"

  type = list(object({
    name  = string
    value = string
  }))

  default  = []
  nullable = false
}

# 创建Kafka主题
resource "huaweicloud_dms_kafka_topic" "test" {
  instance_id      = huaweicloud_dms_kafka_instance.test.id
  name             = var.topic_name
  partitions       = var.topic_partitions
  replicas         = var.topic_replicas
  aging_time       = var.topic_aging_time
  sync_replication = var.topic_sync_replication
  sync_flushing    = var.topic_sync_flushing
  description      = var.topic_description

  dynamic "configs" {
    for_each = var.topic_configs

    content {
      name  = configs.value.name
      value = configs.value.value
    }
  }
}
```

**参数说明**：
- **instance_id**：Kafka实例ID，通过引用Kafka实例资源进行赋值
- **name**：主题名称，通过引用输入变量topic_name进行赋值
- **partitions**：分区数量，通过引用输入变量topic_partitions进行赋值，默认值为10
- **replicas**：副本数量，通过引用输入变量topic_replicas进行赋值，默认值为3
- **aging_time**：老化时间，通过引用输入变量topic_aging_time进行赋值，默认值为72（小时）
- **sync_replication**：同步复制，通过引用输入变量topic_sync_replication进行赋值，默认值为false
- **sync_flushing**：同步刷新，通过引用输入变量topic_sync_flushing进行赋值，默认值为false
- **description**：主题描述，通过引用输入变量topic_description进行赋值，可选参数
- **configs**：主题配置，通过动态块进行配置，可选参数

### 6. 创建Kafka消息生产资源

在TF文件（如main.tf）中添加以下脚本以创建Kafka消息生产资源，将消息发送到主题：

```hcl
variable "message_body" {
  description = "The body of the message to be sent"
  type        = string
}

variable "message_properties" {
  description = "The properties of the message to be sent"

  type = list(object({
    name  = string
    value = string
  }))

  default  = []
  nullable = false
}

# 创建Kafka消息生产
resource "huaweicloud_dms_kafka_message_produce" "test" {
  instance_id = huaweicloud_dms_kafka_instance.test.id
  topic       = huaweicloud_dms_kafka_topic.test.name
  body        = var.message_body

  dynamic "property_list" {
    for_each = var.message_properties

    content {
      name  = property_list.value.name
      value = property_list.value.value
    }
  }
}
```

**参数说明**：
- **instance_id**：Kafka实例ID，通过引用Kafka实例资源进行赋值
- **topic**：主题名称，通过引用Kafka主题资源进行赋值
- **body**：消息体内容，通过引用输入变量message_body进行赋值
- **property_list**：消息属性列表，通过动态块进行配置，可选参数，支持设置KEY、PARTITION等属性

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC和子网配置
vpc_name    = "tf_test_kafka_instance"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_kafka_instance"

# 安全组配置
security_group_name = "tf_test_kafka_instance"

# Kafka实例配置
instance_name = "tf_test_kafka_instance"

# Kafka主题配置
topic_name = "tf_test_topic"

# 消息生产配置
message_body = "Hello Kafka!"

message_properties = [
  {
    name  = "KEY"
    value = "testKey"
  },
  {
    name  = "PARTITION"
    value = "1"
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `message_body`需要设置要发送的消息内容
   - `message_properties`可以设置消息的属性，如KEY（消息键）、PARTITION（分区号）等
   - 如果Kafka实例启用了认证，需要配置`instance_access_user_name`和`instance_access_user_password`
   - `instance_access_user_password`需要设置符合密码复杂度要求的密码
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="message_body=Hello World" -var="topic_name=my_topic"`
2. 环境变量：`export TF_VAR_message_body=Hello World` 和 `export TF_VAR_topic_name=my_topic`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。由于password包含敏感信息，建议使用环境变量或加密的变量文件进行设置。另外，确保Kafka实例已创建完成且状态正常，主题已创建完成，才能成功生产消息。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建Kafka主题消息生产：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Kafka实例、主题和消息生产
4. 运行 `terraform show` 查看已创建的消息生产详情

> 注意：消息生产资源创建后，消息会立即发送到指定的Kafka主题。如果设置了消息属性（如PARTITION），消息会发送到指定的分区。实例的可用区和规格ID在创建后不能修改，因此需要在创建时正确配置。通过lifecycle.ignore_changes可以避免Terraform在后续更新时修改这些不可变参数。

## 参考信息

- [华为云分布式消息服务Kafka产品文档](https://support.huaweicloud.com/kafka/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [主题消息生产最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/topic-message-produce)
