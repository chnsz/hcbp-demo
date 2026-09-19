# 部署事件订阅（OBS事件源、Kafka事件目标）

## 应用场景

事件网格（EventGrid，EG）是华为云提供的事件驱动架构服务，支持事件的生产、路由、转换和消费。通过事件订阅，您可以将云服务产生的事件实时路由到指定的目标端，实现系统间的松耦合集成。

本最佳实践将介绍如何使用Terraform自动化部署一条事件订阅，以OBS事件源和Kafka事件目标为例，实现OBS存储操作事件的实时推送与Kafka消息队列的集成。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Kafka规格列表（data.huaweicloud_dms_kafka_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)
- [事件通道列表（data.huaweicloud_eg_event_channels）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/eg_event_channels)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [对象存储桶（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [Kafka实例（huaweicloud_dms_kafka_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)
- [Kafka主题（huaweicloud_dms_kafka_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_topic)
- [EG连接（huaweicloud_eg_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_connection)
- [EG事件订阅（huaweicloud_eg_event_subscription）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_event_subscription)
- [OBS桶对象（huaweicloud_obs_bucket_object）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket_object)
- [时间等待（time_sleep）](https://registry.terraform.io/providers/hashicorp/time/latest/docs/resources/sleep)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_dms_kafka_flavors
        └── huaweicloud_dms_kafka_instance
            ├── huaweicloud_dms_kafka_topic
            │   └── huaweicloud_eg_connection
            │       └── time_sleep
            │           └── huaweicloud_eg_event_subscription
            └── huaweicloud_eg_connection
                └── time_sleep
                    └── huaweicloud_eg_event_subscription

data.huaweicloud_eg_event_channels
    └── huaweicloud_eg_event_subscription

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    │   └── huaweicloud_dms_kafka_instance
    └── huaweicloud_eg_connection

huaweicloud_networking_secgroup
    └── huaweicloud_dms_kafka_instance

huaweicloud_obs_bucket
    └── huaweicloud_obs_bucket_object
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云资源
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "172.16.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：通过引用输入变量 vpc_name 进行赋值
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值

### 3. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网资源
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

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway
}
```

**参数说明**：
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值
- **gateway_ip**：通过引用输入变量 subnet_gateway 进行赋值

### 4. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
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

### 5. 创建对象存储桶

在TF文件（如main.tf）中添加以下脚本以创建对象存储桶：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建对象存储桶资源
variable "bucket_name" {
  description = "The name of the OBS bucket"
  type        = string
}

variable "bucket_acl" {
  description = "The ACL policy for a bucket"
  type        = string
  default     = "private"
}

resource "huaweicloud_obs_bucket" "test" {
  bucket        = var.bucket_name
  acl           = var.bucket_acl
  force_destroy = true
}
```

**参数说明**：
- **bucket**：通过引用输入变量 bucket_name 进行赋值
- **acl**：通过引用输入变量 bucket_acl 进行赋值
- **force_destroy**：设置为 true，表示删除桶时强制删除桶内对象

### 6. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询可用区列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表数据源
variable "availability_zones" {
  description = "The availability zones to which the Kafka instance belongs"
  type        = list(string)
  default     = []
}

data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**参数说明**：
- **count**：当输入变量 availability_zones 为空时创建该数据源，用于自动获取可用区列表

### 7. 查询Kafka规格列表

在TF文件（如main.tf）中添加以下脚本以查询Kafka规格列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询Kafka规格列表数据源
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

data "huaweicloud_dms_kafka_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  type               = var.instance_flavor_type
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3)) : var.availability_zones
  storage_spec_code  = var.instance_storage_spec_code
}
```

**参数说明**：
- **count**：当输入变量 instance_flavor_id 为空时创建该数据源，用于自动获取Kafka规格列表
- **type**：通过引用输入变量 instance_flavor_type 进行赋值
- **availability_zones**：当输入变量 availability_zones 为空时，取可用区列表数据源的前三个可用区
- **storage_spec_code**：通过引用输入变量 instance_storage_spec_code 进行赋值

### 8. 创建Kafka实例

在TF文件（如main.tf）中添加以下脚本以创建Kafka实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Kafka实例资源
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
- **name**：通过引用输入变量 instance_name 进行赋值
- **availability_zones**：当输入变量 availability_zones 为空时，取可用区列表数据源的前三个可用区
- **engine_version**：通过引用输入变量 instance_engine_version 进行赋值
- **flavor_id**：当输入变量 instance_flavor_id 为空时，取Kafka规格列表数据源的第一个规格ID
- **storage_spec_code**：通过引用输入变量 instance_storage_spec_code 进行赋值
- **storage_space**：通过引用输入变量 instance_storage_space 进行赋值
- **broker_num**：通过引用输入变量 instance_broker_num 进行赋值
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **network_id**：通过引用虚拟私有云子网资源的ID进行赋值
- **security_group_id**：通过引用安全组资源的ID进行赋值
- **ssl_enable**：通过引用输入变量 instance_ssl_enable 进行赋值
- **description**：通过引用输入变量 instance_description 进行赋值
- **security_protocol**：通过引用输入变量 instance_security_protocol 进行赋值
- **charging_mode**：通过引用输入变量 charging_mode 进行赋值

### 9. 创建Kafka主题

在TF文件（如main.tf）中添加以下脚本以创建Kafka主题：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Kafka主题资源
variable "topic_name" {
  description = "The name of the topic"
  type        = string
}

variable "topic_partitions" {
  description = "The number of the topic partition"
  type        = number
  default     = 3
}

resource "huaweicloud_dms_kafka_topic" "test" {
  instance_id = huaweicloud_dms_kafka_instance.test.id
  name        = var.topic_name
  partitions  = var.topic_partitions
}
```

**参数说明**：
- **instance_id**：通过引用Kafka实例资源的ID进行赋值
- **name**：通过引用输入变量 topic_name 进行赋值
- **partitions**：通过引用输入变量 topic_partitions 进行赋值

### 10. 创建EG连接

在TF文件（如main.tf）中添加以下脚本以创建EG连接：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建EG连接资源
variable "connection_name" {
  description = "The name of the connection"
  type        = string
}

variable "connection_acks" {
  description = "The number of confirmation signals the prouder needs to receive to consider the message sent successfully"
  type        = string
  default     = "1"
}

locals {
  kafka_connect_with_port = join(",", formatlist("%s:${huaweicloud_dms_kafka_instance.test.port}", split(",", huaweicloud_dms_kafka_instance.test.connect_address)))
}

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
- **name**：通过引用输入变量 connection_name 进行赋值
- **type**：连接类型，此处固定为 KAFKA
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **subnet_id**：通过引用虚拟私有云子网资源的ID进行赋值
- **kafka_detail.instance_id**：通过引用Kafka实例资源的ID进行赋值
- **kafka_detail.connect_address**：通过本地变量 kafka_connect_with_port 进行赋值，将Kafka实例的连接地址与端口拼接
- **kafka_detail.acks**：通过引用输入变量 connection_acks 进行赋值

### 11. 等待Kafka主题与EG连接就绪

在TF文件（如main.tf）中添加以下脚本以等待Kafka主题与EG连接就绪：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建时间等待资源
resource "time_sleep" "test" {
  create_duration = "5s"

  depends_on = [
    huaweicloud_eg_connection.test
  ]
}
```

**参数说明**：
- **create_duration**：等待时长，此处设置为 5s，用于等待Kafka主题与EG连接就绪

### 12. 查询事件通道列表

在TF文件（如main.tf）中添加以下脚本以查询事件通道列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询事件通道列表数据源
data "huaweicloud_eg_event_channels" "test" {
  provider_type = "OFFICIAL"
  name          = "default"
}
```

**参数说明**：
- **provider_type**：通道提供方类型，此处固定为 OFFICIAL
- **name**：通道名称，此处固定为 default

### 13. 创建EG事件订阅

在TF文件（如main.tf）中添加以下脚本以创建EG事件订阅：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建EG事件订阅资源
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
- **channel_id**：通过引用事件通道列表数据源中第一个通道的ID进行赋值
- **name**：通过引用事件通道列表数据源中第一个通道的名称进行赋值
- **sources.name**：事件源名称，此处固定为 HC.OBS
- **sources.provider_type**：事件源提供方类型，此处固定为 OFFICIAL
- **sources.filter_rule**：事件过滤规则，按事件源与事件类型进行过滤，事件类型通过引用输入变量 subscription_source_values 进行赋值
- **targets.name**：事件目标名称，此处固定为 HC.Kafka
- **targets.provider_type**：事件目标提供方类型，此处固定为 OFFICIAL
- **targets.connection_id**：通过引用EG连接资源的ID进行赋值
- **targets.transform**：事件转换规则，此处固定为 ORIGINAL
- **targets.detail_name**：事件目标详情名称，此处固定为 kafka_detail
- **targets.detail**：事件目标详情，其中 topic 通过引用Kafka主题资源的名称进行赋值

### 14. 上传OBS桶对象

在TF文件（如main.tf）中添加以下脚本以上传OBS桶对象：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建OBS桶对象资源
variable "object_name" {
  description = "The name of the OBS object to be uploaded"
  type        = string
}

variable "object_extension_name" {
  description = "The extension name of the OBS object to be uploaded"
  type        = string
  default     = ".txt"
  nullable    = false
}

variable "object_upload_content" {
  description = "The content of the OBS object to be uploaded"
  type        = string
}

resource "huaweicloud_obs_bucket_object" "test" {
  bucket       = huaweicloud_obs_bucket.test.id
  key          = var.object_extension_name != "" ? format("%s%s", var.object_name, var.object_extension_name) : var.object_name
  content_type = "application/xml"
  content      = var.object_upload_content
}
```

**参数说明**：
- **bucket**：通过引用对象存储桶资源的ID进行赋值
- **key**：对象名称，当输入变量 object_extension_name 不为空时，将 object_name 与 object_extension_name 拼接
- **content_type**：对象内容类型，此处固定为 application/xml
- **content**：通过引用输入变量 object_upload_content 进行赋值

### 15. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name              = "tf_test_vpc"
subnet_name           = "tf_test_subnet"
security_group_name   = "tf_test_security_group"
bucket_name           = "tf-test-bucket"
instance_name         = "tf_test_kafka"
topic_name            = "tf-test-topic"
connection_name       = "tf-test-connect"
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

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 16. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建事件订阅
4. 运行 `terraform show` 查看已创建的事件订阅

## 参考信息

- [华为云事件网格产品文档](https://support.huaweicloud.com/eg/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EG事件订阅（OBS事件源、Kafka事件目标）最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eg/event-subscriptions/obs-to-kafka)
