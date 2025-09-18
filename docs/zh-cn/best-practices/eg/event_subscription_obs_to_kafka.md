# 部署事件订阅（OBS事件源、Kafka事件目标）

## 应用场景

华为云事件网格（EventGrid，EG）的事件订阅功能支持将OBS（对象存储服务）产生的事件自动路由到Kafka消息队列中，实现云服务间的事件驱动架构。通过这种集成方式，您可以构建实时数据处理管道，将OBS的存储操作事件（如对象创建、删除、修改等）实时推送到Kafka集群，供下游消费者进行实时处理和分析。

本最佳实践特别适用于需要实时监控OBS存储操作、构建数据湖事件驱动架构、实现存储事件与消息队列集成的场景，如数据备份监控、文件处理工作流、实时数据分析等。本最佳实践将介绍如何使用Terraform自动化部署一个完整的OBS事件源到Kafka事件目标的事件订阅配置，包括VPC网络、OBS存储桶、Kafka实例、EG连接和事件订阅的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DMS Kafka规格查询数据源（data.huaweicloud_dms_kafka_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)
- [事件网格事件通道查询数据源（data.huaweicloud_eg_event_channels）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/eg_event_channels)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [OBS存储桶资源（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [DMS Kafka实例资源（huaweicloud_dms_kafka_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)
- [DMS Kafka主题资源（huaweicloud_dms_kafka_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_topic)
- [事件网格连接资源（huaweicloud_eg_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_connection)
- [事件订阅资源（huaweicloud_eg_event_subscription）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_event_subscription)
- [OBS对象资源（huaweicloud_obs_bucket_object）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket_object)
- [时间等待资源（time_sleep）](https://registry.terraform.io/providers/hashicorp/time/latest/docs/resources/sleep)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dms_kafka_instance

data.huaweicloud_dms_kafka_flavors
    └── huaweicloud_dms_kafka_instance

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    ├── huaweicloud_dms_kafka_instance
    └── huaweicloud_eg_connection

huaweicloud_vpc_subnet
    ├── huaweicloud_dms_kafka_instance
    └── huaweicloud_eg_connection

huaweicloud_networking_secgroup
    └── huaweicloud_dms_kafka_instance

huaweicloud_dms_kafka_instance
    ├── huaweicloud_dms_kafka_topic
    └── huaweicloud_eg_connection

huaweicloud_dms_kafka_topic
    └── huaweicloud_eg_connection

huaweicloud_eg_connection
    └── time_sleep
        └── huaweicloud_eg_event_subscription

data.huaweicloud_eg_event_channels
    └── huaweicloud_eg_event_subscription

huaweicloud_obs_bucket
    └── huaweicloud_obs_bucket_object
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC网络

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
# Variable definitions for VPC
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "172.16.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC的名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值，默认值为"172.16.0.0/16"

### 3. 创建VPC子网

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
# Variable definitions for subnet
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "172.16.10.0/24"
}

variable "subnet_gateway" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = "172.16.10.1"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR块，通过引用输入变量subnet_cidr进行赋值，默认值为"172.16.10.0/24"
- **gateway_ip**：子网的网关IP地址，通过引用输入变量subnet_gateway进行赋值，默认值为"172.16.10.1"

### 4. 创建安全组

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
# Variable definitions for security group
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**参数说明**：
- **name**：安全组的名称，通过引用输入变量security_group_name进行赋值

### 5. 创建OBS存储桶

在TF文件中添加以下脚本以告知Terraform创建OBS存储桶资源：

```hcl
# Variable definitions for OBS bucket
variable "bucket_name" {
  description = "The name of the OBS bucket"
  type        = string
}

variable "bucket_acl" {
  description = "The ACL policy for a bucket"
  type        = string
  default     = "private"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建OBS存储桶资源
resource "huaweicloud_obs_bucket" "test" {
  bucket        = var.bucket_name
  acl           = var.bucket_acl
  force_destroy = true
}
```

**参数说明**：
- **bucket**：OBS存储桶的名称，通过引用输入变量bucket_name进行赋值
- **acl**：存储桶的ACL策略，通过引用输入变量bucket_acl进行赋值，默认值为"private"
- **force_destroy**：是否强制删除存储桶，设置为true表示允许删除非空存储桶

### 6. 查询可用区信息

在TF文件中添加以下脚本以告知Terraform查询可用区信息：

```hcl
# Variable definitions for availability zones
variable "availability_zones" {
  description = "The availability zones to which the Kafka instance belongs"
  type        = list(string)
  default     = []
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有可用的可用区信息，用于创建Kafka实例
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**参数说明**：
- **count**：条件创建，当availability_zones变量为空列表时创建此数据源

### 7. 查询DMS Kafka规格信息

在TF文件中添加以下脚本以告知Terraform查询DMS Kafka规格信息：

```hcl
# Variable definitions for Kafka flavors
variable "instance_flavor_id" {
  description = "The flavor ID of the Kafka instance"
  type        = string
  default     = "kafka.2u4g.cluster.small"
}

variable "instance_flavor_type" {
  description = "The flavor type of the Kafka instance"
  type        = string
  default     = "cluster"
}

variable "instance_storage_spec_code" {
  description = "The storage specification code of the Kafka instance"
  type        = string
  default     = "dms.physical.storage.high.v2"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的DMS Kafka规格信息，用于创建Kafka实例
data "huaweicloud_dms_kafka_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  type               = var.instance_flavor_type
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3)) : var.availability_zones
  storage_spec_code  = var.instance_storage_spec_code
}
```

**参数说明**：
- **count**：条件创建，当instance_flavor_id变量为空字符串时创建此数据源
- **type**：规格类型，通过引用输入变量instance_flavor_type进行赋值，默认值为"cluster"
- **availability_zones**：可用区列表，优先使用availability_zones变量，如果为空则使用查询到的可用区
- **storage_spec_code**：存储规格代码，通过引用输入变量instance_storage_spec_code进行赋值，默认值为"dms.physical.storage.high.v2"

### 8. 创建DMS Kafka实例

在TF文件中添加以下脚本以告知Terraform创建DMS Kafka实例资源：

```hcl
# Variable definitions for Kafka instance
variable "instance_name" {
  description = "The name of the Kafka instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the Kafka instance"
  type        = string
  default     = "3.x"
}

variable "instance_storage_space" {
  description = "The storage space of the Kafka instance"
  type        = number
  default     = 300
}

variable "instance_broker_num" {
  description = "The number of brokers of the Kafka instance"
  type        = number
  default     = 3
}

variable "instance_ssl_enable" {
  description = "The SSL enable of the Kafka instance"
  type        = bool
  default     = false
}

variable "instance_description" {
  description = "The description of the Kafka instance"
  type        = string
  default     = ""
}

variable "instance_security_protocol" {
  description = "The protocol to use after SASL is enabled"
  type        = string
  default     = "SASL_SSL"
}

variable "charging_mode" {
  description = "The charging mode of the Kafka instance"
  type        = string
  default     = "postPaid"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DMS Kafka实例资源
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
  description        = var.instance_description
  security_protocol  = var.instance_security_protocol
  charging_mode      = var.charging_mode

  lifecycle {
    ignore_changes = [
      availability_zones,
      flavor_id,
    ]
  }
}
```

**参数说明**：
- **name**：Kafka实例的名称，通过引用输入变量instance_name进行赋值
- **availability_zones**：实例所属的可用区列表，优先使用availability_zones变量，如果为空则使用查询到的可用区
- **engine_version**：Kafka引擎版本，通过引用输入变量instance_engine_version进行赋值，默认值为"3.x"
- **flavor_id**：实例规格ID，优先使用instance_flavor_id变量，如果为空则使用查询到的规格ID
- **storage_spec_code**：存储规格代码，通过引用输入变量instance_storage_spec_code进行赋值
- **storage_space**：存储空间大小，通过引用输入变量instance_storage_space进行赋值，默认值为300
- **broker_num**：Broker节点数量，通过引用输入变量instance_broker_num进行赋值，默认值为3
- **vpc_id**：VPC ID，引用前面创建的VPC资源的ID
- **network_id**：子网ID，引用前面创建的VPC子网资源的ID
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **ssl_enable**：是否启用SSL，通过引用输入变量instance_ssl_enable进行赋值，默认值为false
- **description**：实例描述，通过引用输入变量instance_description进行赋值，默认值为空字符串
- **security_protocol**：安全协议，通过引用输入变量instance_security_protocol进行赋值，默认值为"SASL_SSL"
- **charging_mode**：计费模式，通过引用输入变量charging_mode进行赋值，默认值为"postPaid"
- **lifecycle.ignore_changes**：生命周期管理，忽略availability_zones和flavor_id的变更

### 9. 创建DMS Kafka主题

在TF文件中添加以下脚本以告知Terraform创建DMS Kafka主题资源：

```hcl
# Variable definitions for Kafka topic
variable "topic_name" {
  description = "The name of the topic"
  type        = string
}

variable "topic_partitions" {
  description = "The number of the topic partition"
  type        = number
  default     = 3
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DMS Kafka主题资源
resource "huaweicloud_dms_kafka_topic" "test" {
  instance_id = huaweicloud_dms_kafka_instance.test.id
  name        = var.topic_name
  partitions  = var.topic_partitions
}
```

**参数说明**：
- **instance_id**：Kafka实例ID，引用前面创建的DMS Kafka实例资源的ID
- **name**：主题名称，通过引用输入变量topic_name进行赋值
- **partitions**：主题分区数量，通过引用输入变量topic_partitions进行赋值，默认值为3

### 10. 创建本地变量

在TF文件中添加以下脚本以创建本地变量：

```hcl
# 创建本地变量，用于构建Kafka连接地址
locals {
  kafka_connect_with_port = join(
    ",",
    formatlist(
      "%s:${huaweicloud_dms_kafka_instance.test.port}",
      split(",", huaweicloud_dms_kafka_instance.test.connect_address)
    )
  )
}
```

**参数说明**：
- **kafka_connect_with_port**：包含端口的Kafka连接地址列表，通过格式化Kafka实例的连接地址和端口生成

### 11. 创建事件网格连接

在TF文件中添加以下脚本以告知Terraform创建事件网格连接资源：

```hcl
# Variable definitions for EG connection
variable "connection_name" {
  description = "The name of the connection"
  type        = string
}

variable "connection_acks" {
  description = "The number of confirmation signals the prouder needs to receive to consider the message sent successfully"
  type        = string
  default     = "1"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建事件网格连接资源
resource "huaweicloud_eg_connection" "test" {
  name      = var.connection_name
  type      = "KAFKA"
  vpc_id    = huaweicloud_vpc.test.id
  subnet_id = huaweicloud_vpc_subnet.test.id

  kafka_detail {
    instance_id     = huaweicloud_dms_kafka_instance.test.id
    connect_address = local.kafka_connect_with_port
    acks            = var.connection_acks
  }

  lifecycle {
    ignore_changes = [
      kafka_detail[0].user_name,
      kafka_detail[0].password,
    ]
  }

  depends_on = [
    huaweicloud_dms_kafka_topic.test
  ]
}
```

**参数说明**：
- **name**：连接的名称，通过引用输入变量connection_name进行赋值
- **type**：连接类型，设置为"KAFKA"表示Kafka连接
- **vpc_id**：VPC ID，引用前面创建的VPC资源的ID
- **subnet_id**：子网ID，引用前面创建的VPC子网资源的ID
- **kafka_detail**：Kafka连接详细配置
  - **instance_id**：Kafka实例ID，引用前面创建的DMS Kafka实例资源的ID
  - **connect_address**：连接地址，使用本地变量kafka_connect_with_port的值
  - **acks**：确认信号数量，通过引用输入变量connection_acks进行赋值，默认值为"1"
- **lifecycle.ignore_changes**：生命周期管理，忽略kafka_detail中的用户名和密码变更
- **depends_on**：显式依赖关系，确保Kafka主题在连接创建前已存在

### 12. 创建时间等待资源

在TF文件中添加以下脚本以告知Terraform创建时间等待资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建时间等待资源，用于等待Kafka主题和EG连接准备就绪
resource "time_sleep" "test" {
  create_duration = "5s"

  depends_on = [
    huaweicloud_eg_connection.test
  ]
}
```

**参数说明**：
- **create_duration**：等待时间，设置为"5s"表示等待5秒
- **depends_on**：显式依赖关系，确保事件网格连接在时间等待资源创建前已存在

### 13. 查询事件网格事件通道信息

在TF文件中添加以下脚本以告知Terraform查询事件网格事件通道信息：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的事件网格事件通道信息，用于创建事件订阅
data "huaweicloud_eg_event_channels" "test" {
  provider_type = "OFFICIAL"
  name          = "default"
}
```

**参数说明**：
- **provider_type**：提供者类型，设置为"OFFICIAL"表示官方提供者
- **name**：通道名称，设置为"default"表示默认通道

### 14. 创建事件订阅资源

在TF文件中添加以下脚本以告知Terraform创建事件订阅资源：

```hcl
# Variable definitions for event subscription
variable "subscription_source_values" {
  description = "The event types to be subscribed from OBS service"
  type        = list(string)
  default     = [
    "OBS:CloudTrace:ApiCall",
    "OBS:CloudTrace:ObsSDK",
    "OBS:CloudTrace:ConsoleAction",
    "OBS:CloudTrace:SystemAction",
    "OBS:CloudTrace:Others"
  ]
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建事件订阅资源
resource "huaweicloud_eg_event_subscription" "test" {
  channel_id = try(data.huaweicloud_eg_event_channels.test.channels[0].id, "")
  name       = try(data.huaweicloud_eg_event_channels.test.channels[0].name, "")

  sources {
    name          = "HC.OBS"
    provider_type = "OFFICIAL"

    filter_rule = jsonencode({
      "source" : [
        {
          "op" : "StringIn",
          "values" : ["HC.OBS"]
        }
      ],
      "type" : [
        {
          "op" : "StringIn",
          "values" : var.subscription_source_values
        }
      ],
    })
  }

  targets {
    name          = "HC.Kafka"
    provider_type = "OFFICIAL"
    connection_id = huaweicloud_eg_connection.test.id

    transform = jsonencode({
      "type" : "ORIGINAL",
    })

    detail_name = "kafka_detail"
    detail      = jsonencode({
      "topic" : huaweicloud_dms_kafka_topic.test.name
      "key_transform" : {
        "type" : "ORIGINAL",
      }
    })
  }

  lifecycle {
    ignore_changes = [
      sources
    ]
  }

  depends_on = [
    time_sleep.test
  ]
}
```

**参数说明**：
- **channel_id**：事件通道ID，使用查询到的默认事件通道ID
- **name**：事件订阅名称，使用查询到的默认事件通道名称
- **sources**：事件源配置块
  - **name**：事件源名称，设置为"HC.OBS"表示OBS服务
  - **provider_type**：事件源提供者类型，设置为"OFFICIAL"表示官方提供者
  - **filter_rule**：过滤规则，以JSON格式配置事件源过滤条件，包括source和type过滤
- **targets**：事件目标配置块
  - **name**：事件目标名称，设置为"HC.Kafka"表示Kafka服务
  - **provider_type**：事件目标提供者类型，设置为"OFFICIAL"表示官方提供者
  - **connection_id**：连接ID，引用前面创建的事件网格连接资源的ID
  - **transform**：转换配置，以JSON格式配置事件转换规则，设置为"ORIGINAL"表示保持原始格式
  - **detail_name**：目标详情配置的名称，设置为"kafka_detail"
  - **detail**：目标详情配置，以JSON格式配置Kafka主题和键转换规则
- **lifecycle.ignore_changes**：生命周期管理，忽略sources的变更以避免重建订阅
- **depends_on**：显式依赖关系，确保时间等待资源在事件订阅创建前已完成

### 15. 创建OBS对象

在TF文件中添加以下脚本以告知Terraform创建OBS对象资源：

```hcl
# Variable definitions for OBS object
variable "object_extension_name" {
  description = "The extension name of the OBS object to be uploaded"
  type        = string
  default     = ".txt"
  nullable    = false
}

variable "object_name" {
  description = "The name of the OBS object to be uploaded"
  type        = string
}

variable "object_upload_content" {
  description = "The content of the OBS object to be uploaded"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建OBS对象资源
resource "huaweicloud_obs_bucket_object" "test" {
  bucket       = huaweicloud_obs_bucket.test.id
  key          = var.object_extension_name != "" ? format("%s%s", var.object_name, var.object_extension_name) : var.object_name
  content_type = "application/xml"
  content      = var.object_upload_content
}
```

**参数说明**：
- **bucket**：存储桶ID，引用前面创建的OBS存储桶资源的ID
- **key**：对象键名，根据object_extension_name是否为空决定是否添加扩展名
- **content_type**：对象内容类型，设置为"application/xml"
- **content**：对象内容，通过引用输入变量object_upload_content进行赋值

### 16. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC网络配置
vpc_name    = "tf_test_vpc"
subnet_name = "tf_test_subnet"

# 安全组配置
security_group_name = "tf_test_security_group"

# OBS存储桶配置
bucket_name = "tf-test-bucket"

# Kafka实例配置
instance_name = "tf_test_kafka"

# Kafka主题配置
topic_name = "tf-test-topic"

# 事件网格连接配置
connection_name = "tf-test-connect"

# OBS对象配置
object_name           = "tf-test-obs-object"
object_upload_content = <<EOT
def main():
    print("Hello, World!")

if __name__ == "__main__":
    main()
EOT
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="bucket_name=my-bucket"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 17. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建事件订阅（OBS事件源、Kafka事件目标）
4. 运行 `terraform show` 查看已创建的事件订阅（OBS事件源、Kafka事件目标）详情

## 参考信息

- [华为云事件网格产品文档](https://support.huaweicloud.com/eg/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [事件网格最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eg)
