# 部署Kafka实例数据复制

## 应用场景

分布式消息服务（DMS）Kafka版提供高吞吐、高可靠的消息中间件能力，广泛应用于日志采集、流式数据处理和业务解耦等场景。当业务需要在不同Kafka实例之间同步数据时，可以通过Smart Connect任务实现实例间的数据复制，满足跨实例数据迁移、多活容灾和数据汇聚等需求。

本最佳实践将介绍如何使用Terraform自动化部署Kafka实例数据复制，包括创建VPC、子网、安全组、多个Kafka实例、Kafka主题以及Smart Connect和Smart Connect任务。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Kafka实例规格（data.huaweicloud_dms_kafka_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Kafka实例（huaweicloud_dms_kafka_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)
- [Kafka主题（huaweicloud_dms_kafka_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_topic)
- [Kafka Smart Connect（huaweicloud_dms_kafka_smart_connect）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_smart_connect)
- [Kafka Smart Connect任务（huaweicloud_dms_kafkav2_smart_connect_task）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafkav2_smart_connect_task)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dms_kafka_instance

data.huaweicloud_dms_kafka_flavors
    └── huaweicloud_dms_kafka_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dms_kafka_instance
            ├── huaweicloud_dms_kafka_topic
            ├── huaweicloud_dms_kafka_smart_connect
            └── huaweicloud_dms_kafkav2_smart_connect_task

huaweicloud_networking_secgroup
    └── huaweicloud_dms_kafka_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
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

data "huaweicloud_availability_zones" "test" {
  count = anytrue([for v in var.instance_configurations : length(v.availability_zones) == 0]) ? 1 : 0
}
```

**参数说明**：
- **count**：当任一Kafka实例配置未指定可用区时，才查询可用区列表，用于为实例自动分配可用区

### 3. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本：

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

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网
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
- **vpc_id**：通过引用资源 huaweicloud_vpc.test 的ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，未指定时基于VPC网段自动划分子网网段
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，未指定时基于子网网段自动计算网关IP

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值

### 6. 查询Kafka实例规格

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询Kafka实例规格
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
- **count**：未指定规格ID的Kafka实例配置数量
- **type**：通过引用实例配置中的 flavor_type 进行赋值
- **availability_zones**：通过引用可用区列表数据源的结果进行赋值
- **storage_spec_code**：通过引用实例配置中的 storage_spec_code 进行赋值

### 7. 创建Kafka实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Kafka实例
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
- **count**：Kafka实例配置数量，至少为2个
- **name**：通过引用实例配置中的 name 进行赋值
- **availability_zones**：通过引用实例配置中的 availability_zones 或可用区列表数据源的结果进行赋值
- **engine_version**：通过引用实例配置中的 engine_version 进行赋值
- **flavor_id**：通过引用实例配置中的 flavor_id 或Kafka实例规格数据源的结果进行赋值
- **storage_spec_code**：通过引用实例配置中的 storage_spec_code 进行赋值
- **storage_space**：通过引用实例配置中的 storage_space 进行赋值
- **broker_num**：通过引用实例配置中的 broker_num 进行赋值
- **vpc_id**：通过引用资源 huaweicloud_vpc.test 的ID进行赋值
- **network_id**：通过引用资源 huaweicloud_vpc_subnet.test 的ID进行赋值
- **security_group_id**：通过引用资源 huaweicloud_networking_secgroup.test 的ID进行赋值
- **access_user**：通过引用实例配置中的 access_user 进行赋值
- **password**：通过引用实例配置中的 password 进行赋值
- **enabled_mechanisms**：通过引用实例配置中的 enabled_mechanisms 进行赋值
- **port_protocol**：通过引用实例配置中的 port_protocol 进行赋值，用于配置实例的端口协议

### 8. 创建Kafka主题

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Kafka主题
variable "task_topics" {
  description = "The topics of the Smart Connect task"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "topic_name" {
  description = "The name of the Kafka topic"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.topic_name != "" || length(var.task_topics) > 0
    error_message = "topic_name is required when task_topics is not provided"
  }
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
- **count**：当未指定Smart Connect任务主题时创建主题
- **instance_id**：通过引用资源 huaweicloud_dms_kafka_instance.test[0] 的ID进行赋值
- **name**：通过引用输入变量 topic_name 进行赋值
- **partitions**：通过引用输入变量 topic_partitions 进行赋值
- **replicas**：通过引用输入变量 topic_replicas 进行赋值
- **aging_time**：通过引用输入变量 topic_aging_time 进行赋值
- **sync_replication**：通过引用输入变量 topic_sync_replication 进行赋值
- **sync_flushing**：通过引用输入变量 topic_sync_flushing 进行赋值
- **description**：通过引用输入变量 topic_description 进行赋值
- **configs**：通过引用输入变量 topic_configs 进行赋值，用于配置主题参数

### 9. 创建Kafka Smart Connect

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Kafka Smart Connect
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

resource "huaweicloud_dms_kafka_smart_connect" "test" {
  instance_id       = huaweicloud_dms_kafka_instance.test[0].id
  storage_spec_code = var.smart_connect_storage_spec_code
  bandwidth         = var.smart_connect_bandwidth
  node_count        = var.smart_connect_node_count
}
```

**参数说明**：
- **instance_id**：通过引用资源 huaweicloud_dms_kafka_instance.test[0] 的ID进行赋值
- **storage_spec_code**：通过引用输入变量 smart_connect_storage_spec_code 进行赋值
- **bandwidth**：通过引用输入变量 smart_connect_bandwidth 进行赋值
- **node_count**：通过引用输入变量 smart_connect_node_count 进行赋值

### 10. 创建Kafka Smart Connect任务

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Kafka Smart Connect任务
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
- **instance_id**：通过引用资源 huaweicloud_dms_kafka_instance.test[0] 的ID进行赋值
- **task_name**：通过引用输入变量 task_name 进行赋值
- **source_type**：Smart Connect任务源类型，固定为 KAFKA_REPLICATOR_SOURCE
- **start_later**：通过引用输入变量 task_start_later 进行赋值
- **topics**：通过引用输入变量 task_topics 或资源 huaweicloud_dms_kafka_topic.test 的名称进行赋值
- **peer_instance_id**：通过引用资源 huaweicloud_dms_kafka_instance.test[1] 的ID进行赋值
- **direction**：通过引用输入变量 task_direction 进行赋值
- **replication_factor**：通过引用输入变量 task_replication_factor 进行赋值
- **task_num**：通过引用输入变量 task_task_num 进行赋值
- **provenance_header_enabled**：通过引用输入变量 task_provenance_header_enabled 进行赋值
- **sync_consumer_offsets_enabled**：通过引用输入变量 task_sync_consumer_offsets_enabled 进行赋值
- **rename_topic_enabled**：通过引用输入变量 task_rename_topic_enabled 进行赋值
- **consumer_strategy**：通过引用输入变量 task_consumer_strategy 进行赋值
- **compression_type**：通过引用输入变量 task_compression_type 进行赋值
- **topics_mapping**：通过引用输入变量 task_topics_mapping 进行赋值
- **security_protocol**：根据对端Kafka实例的端口协议自动选择安全协议
- **sasl_mechanism**：根据对端Kafka实例的SASL机制自动赋值
- **user_name**：根据对端Kafka实例的访问用户自动赋值
- **password**：根据对端Kafka实例的访问密码自动赋值

### 11. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name            = "tf_test_kafka_instance"
subnet_name         = "tf_test_kafka_instance"
security_group_name = "tf_test_kafka_instance"
task_name           = "tf_test_kafka_task"
topic_name          = "tf_test_kafka_topic"

instance_configurations = [
  {
    name = "tf_test_instance"
  },
  {
    name               = "tf_test_peer_instance"
    access_user        = "admin"
    password           = "YourKafkaInstancePassword!"
    enabled_mechanisms = ["SCRAM-SHA-512"]
    port_protocol = {
      private_plain_enable    = false
      private_sasl_ssl_enable = true
    }
  }
]
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

### 12. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Kafka实例数据复制相关资源
4. 运行 `terraform show` 查看已创建的Kafka实例数据复制相关资源

## 参考信息

- [华为云分布式消息服务Kafka产品文档](https://support.huaweicloud.com/kafka/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DMS Kafka实例数据复制最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/replicate-instance-data)
