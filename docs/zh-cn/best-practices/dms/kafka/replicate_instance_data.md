# 部署Kafka实例数据复制

## 应用场景

华为云分布式消息服务Kafka版支持通过Smart Connect实现Kafka实例之间的数据复制，适用于跨地域数据同步、灾备、数据迁移等场景。通过配置Smart Connect任务，可以在源实例和目标实例之间建立数据复制通道，实现主题数据的自动同步。本最佳实践将介绍如何使用Terraform自动化部署Kafka实例数据复制，包括创建多个Kafka实例、Smart Connect和Smart Connect任务。

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
- [Kafka Smart Connect资源（huaweicloud_dms_kafka_smart_connect）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_smart_connect)
- [Kafka Smart Connect任务资源（huaweicloud_dms_kafkav2_smart_connect_task）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafkav2_smart_connect_task)

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
    ├── huaweicloud_dms_kafka_topic
    ├── huaweicloud_dms_kafka_smart_connect
    └── huaweicloud_dms_kafkav2_smart_connect_task
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
  count = anytrue([for v in var.instance_configurations : length(v.availability_zones) == 0]) ? 1 : 0
}

# 查询Kafka规格信息（仅查询未指定flavor_id的实例配置）
locals {
  instance_configurations_without_flavor_id = [for v in var.instance_configurations : v if v.flavor_id == ""]
}

data "huaweicloud_dms_kafka_flavors" "test" {
  count = length(local.instance_configurations_without_flavor_id)

  type               = local.instance_configurations_without_flavor_id[count.index].flavor_type
  availability_zones = length(local.instance_configurations_without_flavor_id[count.index].availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : null
  storage_spec_code  = local.instance_configurations_without_flavor_id[count.index].storage_spec_code
}
```

**参数说明**：
- **type**：规格类型，通过引用本地变量instance_configurations_without_flavor_id进行赋值，默认值为"cluster"（集群模式）
- **availability_zones**：可用区列表，通过引用输入变量或可用区数据源进行赋值
- **storage_spec_code**：存储规格代码，通过引用本地变量进行赋值，默认值为"dms.physical.storage.ultra.v2"

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

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建多个Kafka实例资源（至少需要2个实例）：

```hcl
variable "instance_configurations" {
  description = "The list of configurations for multiple Kafka instances"

  type = list(object({
    name               = string
    availability_zones = optional(list(string), [])
    engine_version     = optional(string, "3.x")
    flavor_id          = optional(string, "")
    flavor_type        = optional(string, "cluster")
    storage_spec_code  = optional(string, "dms.physical.storage.ultra.v2")
    storage_space      = optional(number, 600)
    broker_num         = optional(number, 3)
    access_user        = optional(string, "")
    password           = optional(string, "")
    enabled_mechanisms = optional(list(string), null)

    port_protocol = optional(object({
      private_plain_enable          = optional(bool, true)
      private_sasl_ssl_enable       = optional(bool, null)
      private_sasl_plaintext_enable = optional(bool, null)
    }), {})
  }))

  nullable = false
  default  = []

  validation {
    condition     = length(var.instance_configurations) >= 2
    error_message = "At least two instances are required"
  }
}

# 创建多个Kafka实例（至少需要2个实例用于数据复制）
resource "huaweicloud_dms_kafka_instance" "test" {
  count = length(var.instance_configurations)

  name               = var.instance_configurations[count.index].name
  availability_zones = length(var.instance_configurations[count.index].availability_zones) > 0 ? var.instance_configurations[count.index].availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1))
  engine_version     = var.instance_configurations[count.index].engine_version
  flavor_id          = var.instance_configurations[count.index].flavor_id != "" ? var.instance_configurations[count.index].flavor_id : try(data.huaweicloud_dms_kafka_flavors.test[count.index].flavors[0].id, null)
  storage_spec_code  = var.instance_configurations[count.index].storage_spec_code
  storage_space      = var.instance_configurations[count.index].storage_space
  broker_num         = var.instance_configurations[count.index].broker_num
  vpc_id             = huaweicloud_vpc.test.id
  network_id         = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  access_user        = var.instance_configurations[count.index].access_user
  password           = var.instance_configurations[count.index].password
  enabled_mechanisms = var.instance_configurations[count.index].enabled_mechanisms

  dynamic "port_protocol" {
    for_each = length(var.instance_configurations[count.index].port_protocol) > 0 ? [var.instance_configurations[count.index].port_protocol] : []

    content {
      private_plain_enable          = port_protocol.value.private_plain_enable
      private_sasl_ssl_enable       = port_protocol.value.private_sasl_ssl_enable
      private_sasl_plaintext_enable = port_protocol.value.private_sasl_plaintext_enable
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
- **count**：创建数量，通过引用输入变量instance_configurations的长度进行赋值，至少需要2个实例
- **name**：Kafka实例名称，通过引用输入变量instance_configurations进行赋值
- **availability_zones**：可用区列表，通过引用输入变量或可用区数据源进行赋值
- **engine_version**：引擎版本，通过引用输入变量进行赋值，默认值为"3.x"
- **flavor_id**：规格ID，通过引用输入变量或Kafka规格数据源进行赋值
- **storage_spec_code**：存储规格代码，通过引用输入变量进行赋值，默认值为"dms.physical.storage.ultra.v2"
- **storage_space**：存储空间，通过引用输入变量进行赋值，默认值为600（GB）
- **broker_num**：Broker数量，通过引用输入变量进行赋值，默认值为3
- **vpc_id**：VPC ID，通过引用VPC资源进行赋值
- **network_id**：网络子网ID，通过引用子网资源进行赋值
- **security_group_id**：安全组ID，通过引用安全组资源进行赋值
- **access_user**：访问用户名，通过引用输入变量进行赋值，可选参数
- **password**：访问密码，通过引用输入变量进行赋值，可选参数
- **enabled_mechanisms**：启用的认证机制，通过引用输入变量进行赋值，可选参数，支持"SCRAM-SHA-512"等
- **port_protocol**：端口协议配置，通过动态块进行配置，可选参数

### 5. 创建Kafka主题资源（可选）

在TF文件（如main.tf）中添加以下脚本以创建Kafka主题（如果未指定task_topics，则创建主题）：

```hcl
variable "task_topics" {
  description = "The topics of the Smart Connect task"
  type        = list(string)
  default     = []
}

variable "topic_name" {
  description = "The name of the Kafka topic"
  type        = string
  default     = ""
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

# 创建Kafka主题（仅在未指定task_topics时创建）
resource "huaweicloud_dms_kafka_topic" "test" {
  count = length(var.task_topics) == 0 ? 1 : 0

  instance_id      = huaweicloud_dms_kafka_instance.test[0].id
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
- **count**：创建数量，当task_topics为空时创建1个主题，否则不创建
- **instance_id**：Kafka实例ID，通过引用第一个Kafka实例资源进行赋值
- **name**：主题名称，通过引用输入变量topic_name进行赋值
- **partitions**：分区数量，通过引用输入变量topic_partitions进行赋值，默认值为10
- **replicas**：副本数量，通过引用输入变量topic_replicas进行赋值，默认值为3
- **aging_time**：老化时间，通过引用输入变量topic_aging_time进行赋值，默认值为72（小时）
- **sync_replication**：同步复制，通过引用输入变量topic_sync_replication进行赋值，默认值为false
- **sync_flushing**：同步刷新，通过引用输入变量topic_sync_flushing进行赋值，默认值为false
- **description**：主题描述，通过引用输入变量topic_description进行赋值，可选参数
- **configs**：主题配置，通过动态块进行配置，可选参数

### 6. 创建Smart Connect资源

在TF文件（如main.tf）中添加以下脚本以创建Smart Connect：

```hcl
variable "smart_connect_storage_spec_code" {
  description = "The storage specification code of the Smart Connect"
  type        = string
  default     = null
}

variable "smart_connect_bandwidth" {
  description = "The bandwidth of the Smart Connect"
  type        = string
  default     = null
}

variable "smart_connect_node_count" {
  description = "The number of nodes of the Smart Connect"
  type        = number
  default     = 2
}

# 创建Smart Connect
resource "huaweicloud_dms_kafka_smart_connect" "test" {
  instance_id       = huaweicloud_dms_kafka_instance.test[0].id
  storage_spec_code = var.smart_connect_storage_spec_code
  bandwidth         = var.smart_connect_bandwidth
  node_count        = var.smart_connect_node_count
}
```

**参数说明**：
- **instance_id**：Kafka实例ID，通过引用第一个Kafka实例资源进行赋值
- **storage_spec_code**：存储规格代码，通过引用输入变量smart_connect_storage_spec_code进行赋值，可选参数
- **bandwidth**：带宽，通过引用输入变量smart_connect_bandwidth进行赋值，可选参数
- **node_count**：节点数量，通过引用输入变量smart_connect_node_count进行赋值，默认值为2

### 7. 创建Smart Connect任务资源

在TF文件（如main.tf）中添加以下脚本以创建Smart Connect任务，实现实例之间的数据复制：

```hcl
variable "task_name" {
  description = "The name of the Smart Connect task"
  type        = string
}

variable "task_start_later" {
  description = "The start later of the Smart Connect task"
  type        = bool
  default     = false
}

variable "task_direction" {
  description = "The direction of the Smart Connect task"
  type        = string
  default     = "two-way"
}

variable "task_replication_factor" {
  description = "The replication factor of the Smart Connect task"
  type        = number
  default     = 3
}

variable "task_task_num" {
  description = "The number of tasks of the Smart Connect task"
  type        = number
  default     = 2
}

variable "task_provenance_header_enabled" {
  description = "The provenance header enabled of the Smart Connect task"
  type        = bool
  default     = false
}

variable "task_sync_consumer_offsets_enabled" {
  description = "The sync consumer offsets enabled of the Smart Connect task"
  type        = bool
  default     = false
}

variable "task_rename_topic_enabled" {
  description = "The rename topic enabled of the Smart Connect task"
  type        = bool
  default     = true
}

variable "task_consumer_strategy" {
  description = "The consumer strategy of the Smart Connect task"
  type        = string
  default     = "latest"
}

variable "task_compression_type" {
  description = "The compression type of the Smart Connect task"
  type        = string
  default     = "none"
}

variable "task_topics_mapping" {
  description = "The topics mapping of the Smart Connect task"
  type        = list(string)
  default     = []
}

# 创建Smart Connect任务
resource "huaweicloud_dms_kafkav2_smart_connect_task" "test" {
  instance_id = huaweicloud_dms_kafka_instance.test[0].id
  task_name   = var.task_name
  source_type = "KAFKA_REPLICATOR_SOURCE"
  start_later = var.task_start_later
  topics      = length(var.task_topics) > 0 ? var.task_topics : huaweicloud_dms_kafka_topic.test[*].name

  source_task {
    peer_instance_id              = huaweicloud_dms_kafka_instance.test[1].id
    direction                     = var.task_direction
    replication_factor            = var.task_replication_factor
    task_num                      = var.task_task_num
    provenance_header_enabled     = var.task_provenance_header_enabled
    sync_consumer_offsets_enabled = var.task_sync_consumer_offsets_enabled
    rename_topic_enabled          = var.task_rename_topic_enabled
    consumer_strategy             = var.task_consumer_strategy
    compression_type              = var.task_compression_type
    topics_mapping                = var.task_topics_mapping
    security_protocol             = try(huaweicloud_dms_kafka_instance.test[1].port_protocol[0].private_sasl_ssl_enable, false) ? "SASL_SSL" : try(huaweicloud_dms_kafka_instance.test[1].port_protocol[0].private_sasl_plaintext_enable, false) ? "PLAINTEXT" : null
    sasl_mechanism                = try(tolist(huaweicloud_dms_kafka_instance.test[1].enabled_mechanisms)[0], null)
    user_name                     = try(huaweicloud_dms_kafka_instance.test[1].access_user, null)
    password                      = try(huaweicloud_dms_kafka_instance.test[1].password, null)
  }

  depends_on = [huaweicloud_dms_kafka_smart_connect.test]
}
```

**参数说明**：
- **instance_id**：Kafka实例ID，通过引用第一个Kafka实例资源进行赋值
- **task_name**：任务名称，通过引用输入变量task_name进行赋值
- **source_type**：源类型，设置为"KAFKA_REPLICATOR_SOURCE"（Kafka复制源）
- **start_later**：是否稍后启动，通过引用输入变量task_start_later进行赋值，默认值为false
- **topics**：主题列表，通过引用输入变量task_topics或主题资源进行赋值
- **source_task.peer_instance_id**：对等实例ID，通过引用第二个Kafka实例资源进行赋值
- **source_task.direction**：复制方向，通过引用输入变量task_direction进行赋值，默认值为"two-way"（双向）
- **source_task.replication_factor**：复制因子，通过引用输入变量task_replication_factor进行赋值，默认值为3
- **source_task.task_num**：任务数量，通过引用输入变量task_task_num进行赋值，默认值为2
- **source_task.provenance_header_enabled**：是否启用来源头，通过引用输入变量task_provenance_header_enabled进行赋值，默认值为false
- **source_task.sync_consumer_offsets_enabled**：是否同步消费者偏移量，通过引用输入变量task_sync_consumer_offsets_enabled进行赋值，默认值为false
- **source_task.rename_topic_enabled**：是否重命名主题，通过引用输入变量task_rename_topic_enabled进行赋值，默认值为true
- **source_task.consumer_strategy**：消费者策略，通过引用输入变量task_consumer_strategy进行赋值，默认值为"latest"
- **source_task.compression_type**：压缩类型，通过引用输入变量task_compression_type进行赋值，默认值为"none"
- **source_task.topics_mapping**：主题映射，通过引用输入变量task_topics_mapping进行赋值，可选参数
- **source_task.security_protocol**：安全协议，根据对等实例的端口协议配置自动判断
- **source_task.sasl_mechanism**：SASL机制，根据对等实例的认证机制自动判断
- **source_task.user_name**：用户名，从对等实例获取
- **source_task.password**：密码，从对等实例获取

### 8. 预设资源部署所需的入参（可选）

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

# Smart Connect任务配置
task_name  = "tf_test_kafka_task"
topic_name = "tf_test_kafka_topic"

# Kafka实例配置（至少需要2个实例）
instance_configurations = [
  {
    name = "tf_test_instance"
  },
  {
    name               = "tf_test_peer_instance"
    access_user        = "admin"
    password           = "YourKafkaInstancePassword!"
    enabled_mechanisms = ["SCRAM-SHA-512"]
    port_protocol      = {
      private_plain_enable    = false
      private_sasl_ssl_enable = true
    }
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `instance_configurations`需要至少配置2个实例，第一个实例作为目标实例，第二个实例作为源实例
   - 如果源实例启用了SASL认证，需要配置`access_user`、`password`和`enabled_mechanisms`
   - `password`需要设置符合密码复杂度要求的密码
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="task_name=my_task" -var="vpc_name=my_vpc"`
2. 环境变量：`export TF_VAR_task_name=my_task` 和 `export TF_VAR_vpc_name=my_vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。由于password包含敏感信息，建议使用环境变量或加密的变量文件进行设置。另外，确保源实例和目标实例的网络连通性，并且源实例的认证配置正确。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建Kafka实例数据复制：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Kafka实例、Smart Connect和Smart Connect任务
4. 运行 `terraform show` 查看已创建的Smart Connect任务详情

> 注意：Smart Connect任务创建后，会根据配置自动开始数据复制。如果设置了`start_later=true`，任务创建后不会立即启动，需要手动启动。实例的可用区和规格ID在创建后不能修改，因此需要在创建时正确配置。通过lifecycle.ignore_changes可以避免Terraform在后续更新时修改这些不可变参数。

## 参考信息

- [华为云分布式消息服务Kafka产品文档](https://support.huaweicloud.com/kafka/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [实例数据复制最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/replicate-instance-data)
