# 部署RabbitMQ消息生产与消费

## 应用场景

分布式消息服务（DMS）RabbitMQ版是华为云提供的高可用、高可靠、高性能的消息中间件服务，完全兼容开源RabbitMQ，支持消息收发、消息存储与消息路由等能力。在微服务与分布式系统中，生产者与消费者通过消息队列实现异步通信、系统解耦与削峰填谷，是构建松耦合架构的常见模式。

本最佳实践将介绍如何使用Terraform自动化部署一套完整的RabbitMQ消息生产与消费场景，包括创建VPC、子网、安全组、RabbitMQ实例、虚拟主机、交换机、队列及交换机与队列的绑定关系，并在两台ECS实例上分别部署生产者与消费者应用，实现消息的自动发送与消费。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RabbitMQ实例规格（data.huaweicloud_dms_rabbitmq_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_rabbitmq_flavors)
- [ECS规格（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像列表（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RabbitMQ实例（huaweicloud_dms_rabbitmq_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_instance)
- [RabbitMQ虚拟主机（huaweicloud_dms_rabbitmq_vhost）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_vhost)
- [RabbitMQ交换机（huaweicloud_dms_rabbitmq_exchange）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_exchange)
- [RabbitMQ队列（huaweicloud_dms_rabbitmq_queue）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_queue)
- [RabbitMQ交换机与队列绑定（huaweicloud_dms_rabbitmq_exchange_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_exchange_associate)
- [弹性云服务器（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_dms_rabbitmq_flavors
    ├── data.huaweicloud_compute_flavors
    └── huaweicloud_dms_rabbitmq_instance
        └── huaweicloud_dms_rabbitmq_vhost
            ├── huaweicloud_dms_rabbitmq_exchange
            ├── huaweicloud_dms_rabbitmq_queue
            └── huaweicloud_dms_rabbitmq_exchange_associate

data.huaweicloud_images_images
    ├── huaweicloud_compute_instance.producer
    └── huaweicloud_compute_instance.consumer

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_dms_rabbitmq_instance
        ├── huaweicloud_compute_instance.producer
        └── huaweicloud_compute_instance.consumer

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    ├── huaweicloud_dms_rabbitmq_instance
    ├── huaweicloud_compute_instance.producer
    └── huaweicloud_compute_instance.consumer
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询当前region下的可用区列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
data "huaweicloud_availability_zones" "test" {}
```

### 3. 查询RabbitMQ实例规格

在TF文件（如main.tf）中添加以下脚本以查询RabbitMQ实例规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询RabbitMQ实例规格
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

data "huaweicloud_dms_rabbitmq_flavors" "test" {
  type               = var.instance_flavor_type
  storage_spec_code  = var.instance_storage_spec_code
  availability_zones = try(slice(data.huaweicloud_availability_zones.test.names, 0, 1), null)
}
```

**参数说明**：
- **type**：通过引用输入变量 instance_flavor_type 进行赋值，表示RabbitMQ实例的规格类型
- **storage_spec_code**：通过引用输入变量 instance_storage_spec_code 进行赋值，表示RabbitMQ实例的存储规格编码
- **availability_zones**：通过引用可用区列表数据源进行赋值，表示RabbitMQ实例所在的可用区

### 4. 查询ECS规格与镜像

在TF文件（如main.tf）中添加以下脚本以查询用于部署生产者与消费者应用的ECS规格和镜像：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询ECS规格与镜像
variable "ecs_image_name" {
  description = "The image name of the ECS instances (Ubuntu)"
  type        = string
  default     = "Ubuntu 20.04 server 64bit"
}

data "huaweicloud_compute_flavors" "test" {
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = "normal"
  cpu_core_count    = 2
  memory_size       = 4
}

data "huaweicloud_images_images" "test" {
  name       = var.ecs_image_name
  visibility = "public"
}
```

**参数说明**：
- **availability_zone**：通过引用可用区列表数据源进行赋值，表示ECS实例所在的可用区
- **performance_type**：表示ECS规格的性能类型，此处为normal
- **cpu_core_count**：表示ECS规格的CPU核数，此处为2
- **memory_size**：表示ECS规格的内存大小，此处为4
- **name**：通过引用输入变量 ecs_image_name 进行赋值，表示ECS镜像的名称
- **visibility**：表示镜像的可见性，此处为public

### 5. 创建VPC

在TF文件（如main.tf）中添加以下脚本以创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC
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
- **name**：通过引用输入变量 vpc_name 进行赋值，表示VPC的名称
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值，表示VPC的网段

### 6. 创建子网

在TF文件（如main.tf）中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建子网
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "192.168.0.0/24"
  nullable    = true
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = "192.168.0.1"
  nullable    = true
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：通过引用VPC资源进行赋值，表示子网所属的VPC
- **name**：通过引用输入变量 subnet_name 进行赋值，表示子网的名称
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，若为空则根据VPC网段自动计算
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，若为空则根据子网网段自动计算

### 7. 创建安全组与安全组规则

在TF文件（如main.tf）中添加以下脚本以创建安全组及其规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组与安全组规则
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

variable "security_group_rule_configurations" {
  description = "The list of security group rule configurations."

  type = list(object({
    direction        = string
    ethertype        = string
    protocol         = string
    port_range_min   = number
    port_range_max   = number
    remote_ip_prefix = string
    description      = string
  }))

  default = [
    {
      direction        = "ingress"
      ethertype        = "IPv4"
      protocol         = "tcp"
      port_range_min   = 5672
      port_range_max   = 5672
      remote_ip_prefix = "192.168.0.0/16"
      description      = "Allow ECS instances to access RabbitMQ"
    },
    {
      direction        = "ingress"
      ethertype        = "IPv4"
      protocol         = "tcp"
      port_range_min   = 22
      port_range_max   = 22
      remote_ip_prefix = "192.168.0.0/16"
      description      = "Allow SSH access"
    }
  ]
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = length(var.security_group_rule_configurations)

  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = lookup(var.security_group_rule_configurations[count.index], "direction", null)
  ethertype         = lookup(var.security_group_rule_configurations[count.index], "ethertype", null)
  protocol          = lookup(var.security_group_rule_configurations[count.index], "protocol", null)
  port_range_min    = lookup(var.security_group_rule_configurations[count.index], "port_range_min", null)
  port_range_max    = lookup(var.security_group_rule_configurations[count.index], "port_range_max", null)
  remote_ip_prefix  = lookup(var.security_group_rule_configurations[count.index], "remote_ip_prefix", null)
  description       = lookup(var.security_group_rule_configurations[count.index], "description", null)
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值，表示安全组的名称
- **security_group_id**：通过引用安全组资源进行赋值，表示安全组规则的所属安全组
- **direction**：通过引用输入变量 security_group_rule_configurations 进行赋值，表示安全组规则的方向
- **ethertype**：通过引用输入变量 security_group_rule_configurations 进行赋值，表示安全组规则的IP协议类型
- **protocol**：通过引用输入变量 security_group_rule_configurations 进行赋值，表示安全组规则的协议类型
- **port_range_min**：通过引用输入变量 security_group_rule_configurations 进行赋值，表示安全组规则的端口范围最小值
- **port_range_max**：通过引用输入变量 security_group_rule_configurations 进行赋值，表示安全组规则的端口范围最大值
- **remote_ip_prefix**：通过引用输入变量 security_group_rule_configurations 进行赋值，表示安全组规则的远端IP地址
- **description**：通过引用输入变量 security_group_rule_configurations 进行赋值，表示安全组规则的描述

### 8. 创建RabbitMQ实例

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
  description = "Whether to enable SSL for the RabbitMQ instance"
  type        = bool
  default     = false
}

variable "instance_access_user_name" {
  description = "The access user of the RabbitMQ instance"
  type        = string
  default     = "admin"
}

variable "instance_password" {
  description = "The access password of the RabbitMQ instance"
  type        = string
  sensitive   = true
  default     = "123456"
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
  description = "The key/value pairs to associate with the RabbitMQ instance"
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
  flavor_id             = try(data.huaweicloud_dms_rabbitmq_flavors.test.flavors[0].id, null)
  vpc_id                = huaweicloud_vpc.test.id
  network_id            = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zones    = try([data.huaweicloud_availability_zones.test.names[0]], null)
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

  lifecycle {
    ignore_changes = [
      flavor_id,
      availability_zones,
    ]
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 instance_name 进行赋值，表示RabbitMQ实例的名称
- **engine_version**：通过引用输入变量 instance_engine_version 进行赋值，表示RabbitMQ实例的引擎版本
- **flavor_id**：通过引用RabbitMQ实例规格数据源进行赋值，表示RabbitMQ实例的规格
- **vpc_id**：通过引用VPC资源进行赋值，表示RabbitMQ实例所在的VPC
- **network_id**：通过引用子网资源进行赋值，表示RabbitMQ实例所在的子网
- **security_group_id**：通过引用安全组资源进行赋值，表示RabbitMQ实例绑定的安全组
- **availability_zones**：通过引用可用区列表数据源进行赋值，表示RabbitMQ实例所在的可用区
- **broker_num**：通过引用输入变量 instance_broker_num 进行赋值，表示RabbitMQ实例的代理数量
- **storage_space**：通过引用输入变量 instance_storage_space 进行赋值，表示RabbitMQ实例的存储空间
- **storage_spec_code**：通过引用输入变量 instance_storage_spec_code 进行赋值，表示RabbitMQ实例的存储规格编码
- **ssl_enable**：通过引用输入变量 instance_ssl_enable 进行赋值，表示是否开启SSL
- **access_user**：通过引用输入变量 instance_access_user_name 进行赋值，表示RabbitMQ实例的访问用户
- **password**：通过引用输入变量 instance_password 进行赋值，表示RabbitMQ实例的访问密码
- **description**：通过引用输入变量 instance_description 进行赋值，表示RabbitMQ实例的描述
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，表示RabbitMQ实例所属的企业项目
- **tags**：通过引用输入变量 instance_tags 进行赋值，表示RabbitMQ实例的标签
- **charging_mode**：通过引用输入变量 charging_mode 进行赋值，表示RabbitMQ实例的计费模式
- **period_unit**：通过引用输入变量 period_unit 进行赋值，表示RabbitMQ实例的计费周期单位
- **period**：通过引用输入变量 period 进行赋值，表示RabbitMQ实例的计费周期
- **auto_renew**：通过引用输入变量 auto_renew 进行赋值，表示RabbitMQ实例是否自动续费

### 9. 创建RabbitMQ虚拟主机

在TF文件（如main.tf）中添加以下脚本以创建RabbitMQ虚拟主机：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RabbitMQ虚拟主机
variable "vhost_name" {
  description = "The name of the RabbitMQ virtual host"
  type        = string
  default     = "app_vhost"
}

resource "huaweicloud_dms_rabbitmq_vhost" "test" {
  instance_id = huaweicloud_dms_rabbitmq_instance.test.id
  name        = var.vhost_name
}
```

**参数说明**：
- **instance_id**：通过引用RabbitMQ实例资源进行赋值，表示虚拟主机所属的RabbitMQ实例
- **name**：通过引用输入变量 vhost_name 进行赋值，表示虚拟主机的名称

### 10. 创建RabbitMQ交换机

在TF文件（如main.tf）中添加以下脚本以创建RabbitMQ交换机：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RabbitMQ交换机
variable "exchange_name" {
  description = "The name of the RabbitMQ exchange"
  type        = string
  default     = "app_exchange"
}

variable "exchange_type" {
  description = "The type of the RabbitMQ exchange"
  type        = string
  default     = "direct"
}

resource "huaweicloud_dms_rabbitmq_exchange" "test" {
  instance_id = huaweicloud_dms_rabbitmq_instance.test.id
  vhost       = var.vhost_name
  name        = var.exchange_name
  type        = var.exchange_type
  auto_delete = false
  durable     = true
  internal    = false

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_dms_rabbitmq_vhost.test
  ]
}
```

**参数说明**：
- **instance_id**：通过引用RabbitMQ实例资源进行赋值，表示交换机所属的RabbitMQ实例
- **vhost**：通过引用输入变量 vhost_name 进行赋值，表示交换机所属的虚拟主机
- **name**：通过引用输入变量 exchange_name 进行赋值，表示交换机的名称
- **type**：通过引用输入变量 exchange_type 进行赋值，表示交换机的类型
- **auto_delete**：表示交换机是否自动删除，此处为false
- **durable**：表示交换机是否持久化，此处为true
- **internal**：表示交换机是否为内部交换机，此处为false

### 11. 创建RabbitMQ队列

在TF文件（如main.tf）中添加以下脚本以创建RabbitMQ队列：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RabbitMQ队列
variable "queue_name" {
  description = "The name of the RabbitMQ queue"
  type        = string
  default     = "app_queue"
}

resource "huaweicloud_dms_rabbitmq_queue" "test" {
  instance_id = huaweicloud_dms_rabbitmq_instance.test.id
  vhost       = var.vhost_name
  name        = var.queue_name
  auto_delete = false
  durable     = true

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_dms_rabbitmq_vhost.test
  ]
}
```

**参数说明**：
- **instance_id**：通过引用RabbitMQ实例资源进行赋值，表示队列所属的RabbitMQ实例
- **vhost**：通过引用输入变量 vhost_name 进行赋值，表示队列所属的虚拟主机
- **name**：通过引用输入变量 queue_name 进行赋值，表示队列的名称
- **auto_delete**：表示队列是否自动删除，此处为false
- **durable**：表示队列是否持久化，此处为true

### 12. 创建RabbitMQ交换机与队列绑定

在TF文件（如main.tf）中添加以下脚本以将交换机与队列进行绑定：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RabbitMQ交换机与队列绑定
resource "huaweicloud_dms_rabbitmq_exchange_associate" "test" {
  instance_id      = huaweicloud_dms_rabbitmq_instance.test.id
  vhost            = var.vhost_name
  exchange         = var.exchange_name
  destination_type = "Queue"
  destination      = var.queue_name
  routing_key      = var.queue_name

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_dms_rabbitmq_vhost.test,
    huaweicloud_dms_rabbitmq_exchange.test,
    huaweicloud_dms_rabbitmq_queue.test
  ]
}
```

**参数说明**：
- **instance_id**：通过引用RabbitMQ实例资源进行赋值，表示绑定所属的RabbitMQ实例
- **vhost**：通过引用输入变量 vhost_name 进行赋值，表示绑定所属的虚拟主机
- **exchange**：通过引用输入变量 exchange_name 进行赋值，表示绑定的交换机名称
- **destination_type**：表示绑定的目标类型，此处为Queue
- **destination**：通过引用输入变量 queue_name 进行赋值，表示绑定的目标队列名称
- **routing_key**：通过引用输入变量 queue_name 进行赋值，表示绑定的路由键

### 13. 创建生产者ECS实例

在TF文件（如main.tf）中添加以下脚本以创建生产者ECS实例，并通过user_data自动部署生产者应用：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建生产者ECS实例
variable "producer_instance_name" {
  description = "The name of the producer ECS instance"
  type        = string
}

variable "eip_type" {
  description = "The type of the ECS EIP"
  type        = string
  default     = "5_bgp"
}

variable "eip_share_type" {
  description = "The share type of the ECS EIP"
  type        = string
  default     = "PER"
}

variable "eip_size" {
  description = "The size of the ECS EIP"
  type        = number
  default     = 5
}

variable "eip_charge_mode" {
  description = "The charge mode of the ECS EIP"
  type        = string
  default     = "traffic"
}

variable "message_interval" {
  description = "The interval in seconds between messages sent by the producer"
  type        = number
  default     = 5
}

resource "huaweicloud_compute_instance" "producer" {
  name              = var.producer_instance_name
  image_id          = try(data.huaweicloud_images_images.test.images[0].id, null)
  flavor_id         = try(data.huaweicloud_compute_flavors.test.flavors[0].id, null)
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  admin_pass        = var.instance_password
  eip_type          = var.eip_type

  security_group_ids = [huaweicloud_networking_secgroup.test.id]

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  bandwidth {
    share_type  = var.eip_share_type
    size        = var.eip_size
    charge_mode = var.eip_charge_mode
  }

  user_data = templatefile("${path.module}/templates/user_data_producer_tpl", {
    producer_script   = file("${path.module}/apps/producer.py")
    rabbitmq_host     = huaweicloud_dms_rabbitmq_instance.test.connect_address
    rabbitmq_user     = var.instance_access_user_name
    rabbitmq_password = var.instance_password
    rabbitmq_vhost    = var.vhost_name
    queue_name        = var.queue_name
    exchange_name     = var.exchange_name
    exchange_type     = var.exchange_type
    routing_key       = var.queue_name
    message_interval  = var.message_interval
  })

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_dms_rabbitmq_vhost.test,
    huaweicloud_dms_rabbitmq_queue.test,
    huaweicloud_dms_rabbitmq_exchange_associate.test
  ]

  lifecycle {
    ignore_changes = [
      image_id,
      flavor_id,
      availability_zone
    ]
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 producer_instance_name 进行赋值，表示生产者ECS实例的名称
- **image_id**：通过引用镜像列表数据源进行赋值，表示生产者ECS实例的镜像
- **flavor_id**：通过引用ECS规格数据源进行赋值，表示生产者ECS实例的规格
- **availability_zone**：通过引用可用区列表数据源进行赋值，表示生产者ECS实例所在的可用区
- **admin_pass**：通过引用输入变量 instance_password 进行赋值，表示生产者ECS实例的登录密码
- **eip_type**：通过引用输入变量 eip_type 进行赋值，表示生产者ECS实例的EIP类型
- **security_group_ids**：通过引用安全组资源进行赋值，表示生产者ECS实例绑定的安全组
- **network**：表示生产者ECS实例的网络配置，通过引用子网资源进行赋值
- **bandwidth**：表示生产者ECS实例的带宽配置，通过引用输入变量 eip_share_type、eip_size 和 eip_charge_mode 进行赋值
- **user_data**：表示生产者ECS实例的初始化脚本，用于自动部署生产者应用

### 14. 创建消费者ECS实例

在TF文件（如main.tf）中添加以下脚本以创建消费者ECS实例，并通过user_data自动部署消费者应用：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建消费者ECS实例
variable "consumer_instance_name" {
  description = "The name of the consumer ECS instance"
  type        = string
}

resource "huaweicloud_compute_instance" "consumer" {
  name              = var.consumer_instance_name
  image_id          = try(data.huaweicloud_images_images.test.images[0].id, null)
  flavor_id         = try(data.huaweicloud_compute_flavors.test.flavors[0].id, null)
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  admin_pass        = var.instance_password
  eip_type          = var.eip_type

  security_group_ids = [huaweicloud_networking_secgroup.test.id]

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  bandwidth {
    share_type  = var.eip_share_type
    size        = var.eip_size
    charge_mode = var.eip_charge_mode
  }

  user_data = templatefile("${path.module}/templates/user_data_consumer_tpl", {
    consumer_script   = file("${path.module}/apps/consumer.py")
    rabbitmq_host     = huaweicloud_dms_rabbitmq_instance.test.connect_address
    rabbitmq_user     = var.instance_access_user_name
    rabbitmq_password = var.instance_password
    rabbitmq_vhost    = var.vhost_name
    queue_name        = var.queue_name
  })

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_dms_rabbitmq_vhost.test,
    huaweicloud_dms_rabbitmq_queue.test,
    huaweicloud_dms_rabbitmq_exchange_associate.test
  ]

  lifecycle {
    ignore_changes = [
      image_id,
      flavor_id,
      availability_zone
    ]
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 consumer_instance_name 进行赋值，表示消费者ECS实例的名称
- **image_id**：通过引用镜像列表数据源进行赋值，表示消费者ECS实例的镜像
- **flavor_id**：通过引用ECS规格数据源进行赋值，表示消费者ECS实例的规格
- **availability_zone**：通过引用可用区列表数据源进行赋值，表示消费者ECS实例所在的可用区
- **admin_pass**：通过引用输入变量 instance_password 进行赋值，表示消费者ECS实例的登录密码
- **eip_type**：通过引用输入变量 eip_type 进行赋值，表示消费者ECS实例的EIP类型
- **security_group_ids**：通过引用安全组资源进行赋值，表示消费者ECS实例绑定的安全组
- **network**：表示消费者ECS实例的网络配置，通过引用子网资源进行赋值
- **bandwidth**：表示消费者ECS实例的带宽配置，通过引用输入变量 eip_share_type、eip_size 和 eip_charge_mode 进行赋值
- **user_data**：表示消费者ECS实例的初始化脚本，用于自动部署消费者应用

### 15. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 网络变量
vpc_name            = "tf_test_vpc_rabbitmq"
subnet_name         = "tf_test_subnet_rabbitmq"
security_group_name = "tf_test_sg_rabbitmq"

# RabbitMQ实例变量
instance_name             = "tf_test_rabbitmq_instance"
instance_access_user_name = "admin"
instance_password         = "YourPassword@123"

# ECS实例变量
producer_instance_name = "tf_test_producer"
consumer_instance_name = "tf_test_consumer"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建RabbitMQ消息生产与消费场景
4. 运行 `terraform show` 查看已创建的RabbitMQ消息生产与消费场景

## 参考信息

- [华为云分布式消息服务RabbitMQ产品文档](https://support.huaweicloud.com/rabbitmq/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DMS RabbitMQ消息生产与消费最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/rabbitmq/message-producer-and-consumer)
