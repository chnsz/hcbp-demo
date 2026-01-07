# 创建Kafka转发插件

## 应用场景

API网关（API Gateway）的Kafka转发插件是一种用于将HTTP API请求异步转发到Kafka主题的插件，可以实现API请求的异步处理和消息队列集成。Kafka转发插件支持配置Kafka连接信息、主题、消息键、重试策略等参数，帮助您实现灵活的API请求转发管理。通过配置Kafka转发插件，可以将API请求转换为Kafka消息，实现请求的异步处理和后续处理流程的解耦，提高系统的可扩展性和可靠性。本最佳实践将介绍如何使用Terraform自动化创建API网关的Kafka转发插件。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DMS Kafka规格查询数据源（data.huaweicloud_dms_kafka_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [API网关实例资源（huaweicloud_apig_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_instance)
- [DMS Kafka实例资源（huaweicloud_dms_kafka_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)
- [DMS Kafka主题资源（huaweicloud_dms_kafka_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_topic)
- [API网关插件资源（huaweicloud_apig_plugin）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_plugin)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_apig_instance
    └── huaweicloud_dms_kafka_instance

data.huaweicloud_dms_kafka_flavors
    └── huaweicloud_dms_kafka_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_apig_instance
        └── huaweicloud_dms_kafka_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_apig_instance
    └── huaweicloud_dms_kafka_instance

huaweicloud_dms_kafka_instance
    ├── huaweicloud_dms_kafka_topic
    └── huaweicloud_apig_plugin

huaweicloud_apig_instance
    └── huaweicloud_apig_plugin
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建API网关实例和DMS Kafka实例：

```hcl
variable "availability_zones" {
  description = "The availability zones to which the instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "availability_zones_count" {
  description = "The number of availability zones to which the instance belongs"
  type        = number
  default     = 1
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建API网关实例和DMS Kafka实例
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) > 0 ? 0 : 1
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zones` 为空时创建数据源（即执行可用区列表查询）

### 3. 创建VPC资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署API网关实例和DMS Kafka实例
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"

### 4. 创建VPC子网资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署API网关实例和DMS Kafka实例
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源（huaweicloud_vpc.test）的ID
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR网段，通过引用输入变量subnet_cidr进行赋值，当值为空字符串时自动计算
- **gateway_ip**：子网的网关IP，通过引用输入变量subnet_gateway_ip进行赋值，当值为空字符串时自动计算

### 5. 创建安全组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署API网关实例和DMS Kafka实例
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true

### 6. 创建API网关实例资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建API网关实例资源：

```hcl
variable "instance_name" {
  description = "The name of the APIG instance"
  type        = string
}

variable "instance_edition" {
  description = "The edition of the APIG instance"
  type        = string
  default     = "BASIC"
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建API网关实例资源
resource "huaweicloud_apig_instance" "test" {
  name                  = var.instance_name
  edition               = var.instance_edition
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  enterprise_project_id = var.enterprise_project_id
  availability_zones    = length(var.availability_zones) > 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, var.availability_zones_count), null)
}
```

**参数说明**：
- **name**：API网关实例名称，通过引用输入变量instance_name进行赋值
- **edition**：API网关实例版本，通过引用输入变量instance_edition进行赋值，默认值为"BASIC"
- **vpc_id**：VPC ID，引用前面创建的VPC资源（huaweicloud_vpc.test）的ID
- **subnet_id**：子网ID，引用前面创建的VPC子网资源（huaweicloud_vpc_subnet.test）的ID
- **security_group_id**：安全组ID，引用前面创建的安全组资源（huaweicloud_networking_secgroup.test）的ID
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null
- **availability_zones**：可用区列表，当输入变量availability_zones不为空时使用该值，否则根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值

### 7. 通过数据源查询DMS Kafka规格信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建DMS Kafka实例：

```hcl
variable "kafka_instance_flavor_id" {
  description = "The flavor ID of the DMS Kafka instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "kafka_instance_flavor_type" {
  description = "The flavor type of the DMS Kafka instance"
  type        = string
  default     = "cluster"
}

variable "kafka_instance_storage_spec_code" {
  description = "The storage spec code of the DMS Kafka instance"
  type        = string
  default     = "dms.physical.storage.high.v2"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下指定类型的DMS Kafka规格信息，用于创建DMS Kafka实例
data "huaweicloud_dms_kafka_flavors" "test" {
  count = var.kafka_instance_flavor_id != "" ? 0 : 1

  type               = var.kafka_instance_flavor_type
  storage_spec_code  = var.kafka_instance_storage_spec_code
  availability_zones = length(var.availability_zones) > 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3))
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行DMS Kafka规格查询数据源，仅当 `var.kafka_instance_flavor_id` 为空时创建数据源（即执行规格查询）
- **type**：规格类型，通过引用输入变量kafka_instance_flavor_type进行赋值，默认值为"cluster"
- **storage_spec_code**：存储规格代码，通过引用输入变量kafka_instance_storage_spec_code进行赋值
- **availability_zones**：可用区列表，当输入变量availability_zones不为空时使用该值，否则根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值

### 8. 创建DMS Kafka实例资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DMS Kafka实例资源：

```hcl
variable "kafka_instance_name" {
  description = "The name of the DMS Kafka instance"
  type        = string
}

variable "kafka_instance_description" {
  description = "The description of the DMS Kafka instance"
  type        = string
  default     = ""
}

variable "kafka_instance_ssl_enable" {
  description = "Whether to enable SSL for the DMS Kafka instance"
  type        = bool
  default     = false
}

variable "kafka_instance_engine_version" {
  description = "The engine version of the DMS Kafka instance"
  type        = string
}

variable "kafka_instance_storage_space" {
  description = "The storage space of the DMS Kafka instance in GB"
  type        = number
}

variable "kafka_instance_broker_num" {
  description = "The number of brokers for the DMS Kafka instance"
  type        = number
}

variable "kafka_charging_mode" {
  description = "The charging mode of the DMS Kafka instance. Options: prePaid, postPaid"
  type        = string
  default     = "prePaid"
}

variable "kafka_period_unit" {
  description = "The period unit of the DMS Kafka instance. Options: month, year"
  type        = string
  default     = "month"
}

variable "kafka_period" {
  description = "The period of the DMS Kafka instance"
  type        = number
  default     = 1
}

variable "kafka_auto_new" {
  description = "Whether to enable auto renewal for the DMS Kafka instance"
  type        = string
  default     = "false"
}

variable "kafka_instance_user_name" {
  description = "The access user name for the DMS Kafka instance"
  type        = string
  sensitive   = true
}

variable "kafka_instance_user_password" {
  description = "The access user password for the DMS Kafka instance"
  type        = string
  sensitive   = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DMS Kafka实例资源
resource "huaweicloud_dms_kafka_instance" "test" {
  name               = var.kafka_instance_name
  description        = var.kafka_instance_description
  availability_zones = length(var.availability_zones) > 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3))
  vpc_id             = huaweicloud_vpc.test.id
  network_id         = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  ssl_enable         = var.kafka_instance_ssl_enable
  flavor_id          = var.kafka_instance_flavor_id != "" ? var.kafka_instance_flavor_id : try(data.huaweicloud_dms_kafka_flavors.test[0].flavors[0].id, null)
  engine_version     = var.kafka_instance_engine_version
  storage_spec_code  = var.kafka_instance_storage_spec_code
  storage_space      = var.kafka_instance_storage_space
  broker_num         = var.kafka_instance_broker_num
  charging_mode      = var.kafka_charging_mode
  period_unit        = var.kafka_period_unit
  period             = var.kafka_period
  auto_renew         = var.kafka_auto_new
  access_user        = var.kafka_instance_user_name
  password           = var.kafka_instance_user_password

  lifecycle {
    ignore_changes = [
      access_user,
      availability_zones,
      flavor_id,
    ]
  }
}
```

**参数说明**：
- **name**：DMS Kafka实例名称，通过引用输入变量kafka_instance_name进行赋值
- **description**：DMS Kafka实例描述，通过引用输入变量kafka_instance_description进行赋值，默认值为空字符串
- **availability_zones**：可用区列表，当输入变量availability_zones不为空时使用该值，否则根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值
- **vpc_id**：VPC ID，引用前面创建的VPC资源（huaweicloud_vpc.test）的ID
- **network_id**：网络ID，引用前面创建的VPC子网资源（huaweicloud_vpc_subnet.test）的ID
- **security_group_id**：安全组ID，引用前面创建的安全组资源（huaweicloud_networking_secgroup.test）的ID
- **ssl_enable**：是否启用SSL，通过引用输入变量kafka_instance_ssl_enable进行赋值，默认值为false
- **flavor_id**：规格ID，当输入变量kafka_instance_flavor_id不为空时使用该值，否则根据DMS Kafka规格查询数据源（data.huaweicloud_dms_kafka_flavors）的返回结果进行赋值
- **engine_version**：引擎版本，通过引用输入变量kafka_instance_engine_version进行赋值
- **storage_spec_code**：存储规格代码，通过引用输入变量kafka_instance_storage_spec_code进行赋值
- **storage_space**：存储空间（单位：GB），通过引用输入变量kafka_instance_storage_space进行赋值
- **broker_num**：Broker数量，通过引用输入变量kafka_instance_broker_num进行赋值
- **charging_mode**：计费模式，通过引用输入变量kafka_charging_mode进行赋值，默认值为"prePaid"
- **period_unit**：计费周期单位，通过引用输入变量kafka_period_unit进行赋值，默认值为"month"
- **period**：计费周期，通过引用输入变量kafka_period进行赋值，默认值为1
- **auto_renew**：是否自动续费，通过引用输入变量kafka_auto_new进行赋值，默认值为"false"
- **access_user**：访问用户名，通过引用输入变量kafka_instance_user_name进行赋值
- **password**：访问密码，通过引用输入变量kafka_instance_user_password进行赋值

### 9. 创建DMS Kafka主题资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DMS Kafka主题资源：

```hcl
variable "kafka_topic_name" {
  description = "The name of the Kafka topic to receive messages"
  type        = string
}

variable "kafka_topic_partitions" {
  description = "The number of partitions for the Kafka topic"
  type        = number
  default     = 1
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DMS Kafka主题资源
resource "huaweicloud_dms_kafka_topic" "test" {
  instance_id = huaweicloud_dms_kafka_instance.test.id
  name        = var.kafka_topic_name
  partitions  = var.kafka_topic_partitions
}
```

**参数说明**：
- **instance_id**：Kafka实例ID，引用前面创建的DMS Kafka实例资源（huaweicloud_dms_kafka_instance.test）的ID
- **name**：主题名称，通过引用输入变量kafka_topic_name进行赋值
- **partitions**：分区数量，通过引用输入变量kafka_topic_partitions进行赋值，默认值为1

### 10. 创建API网关Kafka转发插件资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建API网关Kafka转发插件资源：

```hcl
variable "plugin_name" {
  description = "The name of the Kafka forward plugin"
  type        = string
}

variable "plugin_description" {
  description = "The description of the Kafka forward plugin"
  type        = string
  default     = null
}

variable "kafka_security_protocol" {
  description = "The security protocol for Kafka connection. Options: PLAINTEXT, SASL_PLAINTEXT, SASL_SSL, SSL"
  type        = string
  default     = "PLAINTEXT"
  nullable    = false

  validation {
    condition     = contains(["PLAINTEXT", "SASL_PLAINTEXT", "SASL_SSL", "SSL"], var.kafka_security_protocol)
    error_message = "kafka_security_protocol must be one of: PLAINTEXT, SASL_PLAINTEXT, SASL_SSL, SSL."
  }
}

variable "kafka_message_key" {
  description = "The message key extraction strategy. Can be a static value or a variable expression like $context.requestId"
  type        = string
  default     = ""
}

variable "kafka_max_retry_count" {
  description = "The maximum number of retry attempts for failed message sends"
  type        = number
  default     = 3
}

variable "kafka_retry_backoff" {
  description = "The backoff time in seconds between retries"
  type        = number
  default     = 10
}

variable "kafka_sasl_mechanisms" {
  description = "The SASL mechanism for authentication. Options: PLAIN, SCRAM-SHA-256, SCRAM-SHA-512"
  type        = string
  default     = "PLAIN"

  validation {
    condition     = contains(["PLAIN", "SCRAM-SHA-256", "SCRAM-SHA-512"], var.kafka_sasl_mechanisms)
    error_message = "kafka_sasl_mechanisms must be one of: PLAIN, SCRAM-SHA-256, SCRAM-SHA-512."
  }
}

variable "kafka_sasl_username" {
  description = "The SASL username for authentication (leave empty to use kafka_access_user)"
  type        = string
  default     = ""
  sensitive   = true
  nullable    = false
}

variable "kafka_access_user" {
  description = "The access user for Kafka authentication (used when kafka_sasl_username is empty and security_protocol is not PLAINTEXT)"
  type        = string
  default     = ""
  sensitive   = true
  nullable    = false
}

variable "kafka_sasl_password" {
  description = "The SASL password for authentication (leave empty to use kafka_password)"
  type        = string
  default     = ""
  sensitive   = true
  nullable    = false
}

variable "kafka_password" {
  description = "The password for Kafka authentication (used when kafka_sasl_password is empty and security_protocol is not PLAINTEXT)"
  type        = string
  default     = ""
  sensitive   = true
  nullable    = false
}

variable "kafka_ssl_ca_content" {
  description = "The SSL CA certificate content for SSL/TLS encrypted connections"
  type        = string
  default     = ""
  sensitive   = true
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建API网关Kafka转发插件资源
resource "huaweicloud_apig_plugin" "test" {
  instance_id = huaweicloud_apig_instance.test.id
  name        = var.plugin_name
  description = var.plugin_description
  type        = "kafka_log"

  content = jsonencode({
    broker_list     = var.kafka_security_protocol == "PLAINTEXT" ? (split(",", huaweicloud_dms_kafka_instance.test.port_protocol[0].private_plain_address)) : var.kafka_security_protocol == "SASL_PLAINTEXT" ? (split(",",huaweicloud_dms_kafka_instance.test.port_protocol[0].private_sasl_plaintext_address)) : (split(",", huaweicloud_dms_kafka_instance.test.port_protocol[0].private_sasl_ssl_address))
    topic           = var.kafka_topic_name
    key             = var.kafka_message_key
    max_retry_count = var.kafka_max_retry_count
    retry_backoff   = var.kafka_retry_backoff

    sasl_config = {
      security_protocol = var.kafka_security_protocol
      sasl_mechanisms   = var.kafka_sasl_mechanisms
      sasl_username     = var.kafka_sasl_username != "" ? nonsensitive(var.kafka_sasl_username) : (var.kafka_security_protocol == "PLAINTEXT" ? "" : nonsensitive(var.kafka_access_user))
      sasl_password     = var.kafka_sasl_password != "" ? nonsensitive(var.kafka_sasl_password) : (var.kafka_security_protocol == "PLAINTEXT" ? "" : nonsensitive(var.kafka_password))
      ssl_ca_content    = var.kafka_ssl_ca_content != "" ? nonsensitive(var.kafka_ssl_ca_content) : ""
    }
  })

  lifecycle {
    ignore_changes = [
      content,
    ]
  }
}
```

**参数说明**：
- **instance_id**：API网关实例ID，引用前面创建的API网关实例资源（huaweicloud_apig_instance.test）的ID
- **name**：插件名称，通过引用输入变量plugin_name进行赋值
- **description**：插件描述，通过引用输入变量plugin_description进行赋值，默认值为null
- **type**：插件类型，设置为"kafka_log"（表示Kafka日志转发插件）
- **content**：插件配置内容，通过jsonencode函数将配置对象编码为JSON字符串，其中：
  - **broker_list**：Kafka Broker列表，根据安全协议类型从DMS Kafka实例的port_protocol属性中获取对应的地址列表
  - **topic**：Kafka主题名称，通过引用输入变量kafka_topic_name进行赋值
  - **key**：消息键提取策略，通过引用输入变量kafka_message_key进行赋值，可以是静态值或变量表达式（如$context.requestId）
  - **max_retry_count**：最大重试次数，通过引用输入变量kafka_max_retry_count进行赋值，默认值为3
  - **retry_backoff**：重试退避时间（单位：秒），通过引用输入变量kafka_retry_backoff进行赋值，默认值为10
  - **sasl_config.security_protocol**：安全协议，通过引用输入变量kafka_security_protocol进行赋值
  - **sasl_config.sasl_mechanisms**：SASL机制，通过引用输入变量kafka_sasl_mechanisms进行赋值
  - **sasl_config.sasl_username**：SASL用户名，当kafka_sasl_username不为空时使用该值，否则当安全协议不为PLAINTEXT时使用kafka_access_user
  - **sasl_config.sasl_password**：SASL密码，当kafka_sasl_password不为空时使用该值，否则当安全协议不为PLAINTEXT时使用kafka_password
  - **sasl_config.ssl_ca_content**：SSL CA证书内容，通过引用输入变量kafka_ssl_ca_content进行赋值

### 11. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC配置
vpc_name    = "tf_test_apig_kafka_forward_plugin"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_apig_kafka_forward_plugin"

# 安全组配置
security_group_name = "tf_test_apig_kafka_forward_plugin"

# API网关实例配置
instance_name     = "tf_test_apig_kafka_forward_plugin"
instance_edition  = "BASIC"

# DMS Kafka实例配置
kafka_instance_name              = "tf_test_apig_kafka_forward_plugin"
kafka_instance_engine_version    = "2.7"
kafka_instance_storage_space     = 600
kafka_instance_broker_num        = 3
kafka_instance_user_name         = "user"
kafka_instance_user_password     = "YourPassword123"

# DMS Kafka主题配置
kafka_topic_name     = "tf_test_apig_kafka_forward_plugin"
kafka_topic_partitions = 1

# API网关插件配置
plugin_name        = "tf_test_apig_kafka_forward_plugin"
plugin_description = "This is a Kafka forward plugin created by Terraform"
kafka_security_protocol = "PLAINTEXT"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=test-vpc" -var="instance_name=test-instance"`
2. 环境变量：`export TF_VAR_vpc_name=test-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 12. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建API网关Kafka转发插件
4. 运行 `terraform show` 查看已创建的API网关Kafka转发插件详情

## 参考信息

- [华为云API网关产品文档](https://support.huaweicloud.com/apig/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [API网关Kafka转发插件最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/apig/kafka-forward-plugin)
