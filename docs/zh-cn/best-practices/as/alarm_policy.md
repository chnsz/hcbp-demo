# 部署告警策略

## 应用场景

华为云弹性伸缩服务（Auto Scaling）是一种自动调整计算资源的服务，能够根据业务需求和策略，自动调整弹性计算实例的数量。告警策略是一种基于监控指标的自动扩缩容策略，通过配置CES（云监控服务）告警规则，当监控指标达到设定阈值时，自动触发伸缩策略执行扩缩容操作。本最佳实践将介绍如何使用Terraform自动化部署告警策略，包括VPC网络、安全组、密钥对、AS配置、AS组、SMN主题、CES告警规则和告警策略的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [计算规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [密钥对资源（huaweicloud_kps_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [AS配置资源（huaweicloud_as_configuration）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_configuration)
- [AS组资源（huaweicloud_as_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_group)
- [SMN主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [CES告警规则资源（huaweicloud_ces_alarmrule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarmrule)
- [AS策略资源（huaweicloud_as_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_policy)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── data.huaweicloud_images_images
            └── huaweicloud_as_configuration

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_as_group

huaweicloud_networking_secgroup
    └── huaweicloud_as_group

huaweicloud_kps_keypair
    └── huaweicloud_as_configuration

huaweicloud_as_configuration
    └── huaweicloud_as_group
        ├── huaweicloud_ces_alarmrule
        └── huaweicloud_as_policy

huaweicloud_smn_topic
    └── huaweicloud_ces_alarmrule
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询告警策略资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建告警策略相关资源：

```hcl
variable "instance_flavor_id" {
  description = "弹性伸缩实例的规格ID"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "弹性伸缩实例规格的性能类型"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "弹性伸缩实例规格的CPU核心数"
  type        = number
  default     = 2
}

variable "instance_flavor_memory_size" {
  description = "弹性伸缩实例规格的内存大小"
  type        = number
  default     = 4
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建告警策略相关资源
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
此数据源无需额外参数，默认查询当前区域下所有可用的可用区信息。

### 3. 通过数据源查询告警策略资源创建所需的实例规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的实例规格：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的实例规格信息，用于创建告警策略相关资源
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行实例规格列表查询，仅当 `var.instance_flavor_id` 为空时创建数据源（即执行实例规格列表查询）
- **availability_zone**：实例规格所在的可用区，使用可用区列表查询数据源的第一个可用区
- **performance_type**：性能类型，通过输入变量 `instance_flavor_performance_type` 进行赋值，默认为"normal"表示标准型
- **cpu_core_count**：CPU核心数，通过输入变量 `instance_flavor_cpu_core_count` 进行赋值
- **memory_size**：内存大小（GB），通过输入变量 `instance_flavor_memory_size` 进行赋值

### 4. 通过数据源查询告警策略资源创建所需的镜像

在TF文件中添加以下脚本以告知Terraform查询符合条件的镜像：

```hcl
variable "instance_image_id" {
  description = "弹性伸缩实例的镜像ID"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的镜像信息，用于创建告警策略相关资源
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].ids[0], null)
  visibility = "public"
  os         = "Ubuntu"
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行镜像列表查询，仅当 `var.instance_image_id` 为空时创建数据源（即执行镜像列表查询）
- **flavor_id**：镜像所支持的规格ID，如果指定了实例规格ID则使用该ID，否则使用实例规格列表查询数据源的第一个规格ID
- **visibility**：镜像可见性，设置为"public"表示公共镜像
- **os**：操作系统类型，设置为"Ubuntu"

### 5. 创建VPC资源

在TF文件中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "VPC的名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于为AS组提供网络环境
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC的名称，通过引用输入变量 `vpc_name` 进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量 `vpc_cidr` 进行赋值，默认为"192.168.0.0/16"

### 6. 创建VPC子网资源

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "子网的名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网的CIDR块"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "子网的网关IP"
  type        = string
  default     = ""

  validation {
    condition     = (var.subnet_cidr != "" && var.subnet_gateway_ip != "") || (var.subnet_cidr == "" && var.subnet_gateway_ip == "")
    error_message = "子网的'subnet_cidr'和'subnet_gateway_ip'不允许其中一个为空"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于为AS组提供网络环境
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **name**：子网的名称，通过引用输入变量 `subnet_name` 进行赋值
- **cidr**：子网的CIDR块，如果指定了子网CIDR则使用该值，否则基于VPC的CIDR块通过 `cidrsubnet` 函数自动计算
- **gateway_ip**：子网的网关IP，如果指定了网关IP则使用该值，否则基于子网CIDR通过 `cidrhost` 函数自动计算

### 7. 创建安全组资源

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "安全组的名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于为AS组提供安全防护
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组的名称，通过引用输入变量 `security_group_name` 进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 8. 创建密钥对资源

在TF文件中添加以下脚本以告知Terraform创建密钥对资源：

```hcl
variable "keypair_name" {
  description = "用于访问弹性伸缩实例的密钥对名称"
  type        = string
}

variable "keypair_public_key" {
  description = "用于访问弹性伸缩实例的密钥对的公钥"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建密钥对资源，用于访问弹性伸缩实例
resource "huaweicloud_kps_keypair" "test" {
  name       = var.keypair_name
  public_key = var.keypair_public_key != "" ? var.keypair_public_key : null
}
```

**参数说明**：
- **name**：密钥对的名称，通过引用输入变量 `keypair_name` 进行赋值
- **public_key**：密钥对的公钥，如果指定了公钥则使用该值，否则设置为null（系统将自动生成密钥对）

### 9. 创建AS配置资源

在TF文件中添加以下脚本以告知Terraform创建AS配置资源：

```hcl
variable "configuration_name" {
  description = "AS配置的名称"
  type        = string
}

variable "disk_configurations" {
  description = "弹性伸缩实例的磁盘配置"
  type = list(object({
    disk_type   = string
    volume_type = string
    volume_size = number
  }))

  nullable = false

  validation {
    condition     = length(var.disk_configurations) > 0 && length([for v in var.disk_configurations : v if v.disk_type == "SYS"]) == 1
    error_message = "磁盘配置不允许为空，且只允许一个系统盘"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AS配置资源，用于定义弹性伸缩实例的模板
resource "huaweicloud_as_configuration" "test" {
  scaling_configuration_name = var.configuration_name

  instance_config {
    image    = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
    flavor   = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
    key_name = huaweicloud_kps_keypair.test.id

    dynamic "disk" {
      for_each = var.disk_configurations

      content {
        disk_type   = disk.value.disk_type
        volume_type = disk.value.volume_type
        size        = disk.value.volume_size
      }
    }
  }
}
```

**参数说明**：
- **scaling_configuration_name**：AS配置的名称，通过引用输入变量 `configuration_name` 进行赋值
- **instance_config**：实例配置块，定义弹性伸缩实例的配置
  - **image**：实例镜像ID，如果指定了镜像ID则使用该值，否则使用镜像列表查询数据源的第一个镜像ID
  - **flavor**：实例规格ID，如果指定了规格ID则使用该值，否则使用实例规格列表查询数据源的第一个规格ID
  - **key_name**：密钥对ID，引用密钥对资源（huaweicloud_kps_keypair.test）的ID进行赋值
  - **disk**：磁盘配置块，通过动态块（dynamic block）根据输入变量 `disk_configurations` 创建多个磁盘配置
    - **disk_type**：磁盘类型，通过输入变量 `disk_configurations` 中的 `disk_type` 进行赋值
    - **volume_type**：磁盘类型，通过输入变量 `disk_configurations` 中的 `volume_type` 进行赋值
    - **size**：磁盘大小（GB），通过输入变量 `disk_configurations` 中的 `volume_size` 进行赋值

> 注意：磁盘配置中必须包含且仅包含一个系统盘（disk_type为"SYS"），其他磁盘为数据盘。

### 10. 创建AS组资源

在TF文件中添加以下脚本以告知Terraform创建AS组资源：

```hcl
variable "group_name" {
  description = "AS组的名称"
  type        = string
}

variable "desire_instance_number" {
  description = "AS组中期望的弹性伸缩实例数量"
  type        = number
  default     = 2
}

variable "min_instance_number" {
  description = "AS组中最小的弹性伸缩实例数量"
  type        = number
  default     = 0
}

variable "max_instance_number" {
  description = "AS组中最大的弹性伸缩实例数量"
  type        = number
  default     = 10
}

variable "is_delete_publicip" {
  description = "删除AS组时是否删除弹性伸缩实例的公网IP地址"
  type        = bool
  default     = true
}

variable "is_delete_instances" {
  description = "删除AS组时是否删除弹性伸缩实例"
  type        = bool
  default     = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AS组资源，用于管理弹性伸缩实例
resource "huaweicloud_as_group" "test" {
  scaling_configuration_id = huaweicloud_as_configuration.test.id
  vpc_id                   = huaweicloud_vpc.test.id
  scaling_group_name       = var.group_name
  desire_instance_number   = var.desire_instance_number
  min_instance_number      = var.min_instance_number
  max_instance_number      = var.max_instance_number
  delete_publicip          = var.is_delete_publicip
  delete_instances         = var.is_delete_instances ? "yes" : "no"

  networks {
    id = huaweicloud_vpc_subnet.test.id
  }

  security_groups {
    id = huaweicloud_networking_secgroup.test.id
  }
}
```

**参数说明**：
- **scaling_configuration_id**：AS配置ID，引用AS配置资源（huaweicloud_as_configuration.test）的ID进行赋值
- **vpc_id**：VPC ID，引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **scaling_group_name**：AS组的名称，通过引用输入变量 `group_name` 进行赋值
- **desire_instance_number**：期望的弹性伸缩实例数量，通过引用输入变量 `desire_instance_number` 进行赋值
- **min_instance_number**：最小的弹性伸缩实例数量，通过引用输入变量 `min_instance_number` 进行赋值
- **max_instance_number**：最大的弹性伸缩实例数量，通过引用输入变量 `max_instance_number` 进行赋值
- **delete_publicip**：删除AS组时是否删除弹性伸缩实例的公网IP地址，通过引用输入变量 `is_delete_publicip` 进行赋值
- **delete_instances**：删除AS组时是否删除弹性伸缩实例，通过引用输入变量 `is_delete_instances` 进行赋值，值为"yes"或"no"
- **networks**：网络配置块，定义AS组使用的子网
  - **id**：子网ID，引用VPC子网资源（huaweicloud_vpc_subnet.test）的ID进行赋值
- **security_groups**：安全组配置块，定义AS组使用的安全组
  - **id**：安全组ID，引用安全组资源（huaweicloud_networking_secgroup.test）的ID进行赋值

### 11. 创建SMN主题资源

在TF文件中添加以下脚本以告知Terraform创建SMN主题资源：

```hcl
variable "topic_name" {
  description = "SMN主题的名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN主题资源，用于接收告警通知
resource "huaweicloud_smn_topic" "test" {
  name = var.topic_name
}
```

**参数说明**：
- **name**：SMN主题的名称，通过引用输入变量 `topic_name` 进行赋值

### 12. 创建CES告警规则资源

在TF文件中添加以下脚本以告知Terraform创建CES告警规则资源：

```hcl
variable "alarm_rule_name" {
  description = "CES告警规则的名称"
  type        = string
}

variable "rule_conditions" {
  description = "告警规则的条件"
  type = list(object({
    alarm_level         = optional(number, 2)
    metric_name         = string
    period              = number
    filter              = string
    comparison_operator = string
    suppress_duration   = optional(number, 0)
    value               = number
    count               = number
  }))

  nullable = false

  validation {
    condition     = length(var.rule_conditions) > 0
    error_message = "告警规则条件不允许为空"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CES告警规则资源，用于监控AS组的指标并触发告警
resource "huaweicloud_ces_alarmrule" "test" {
  alarm_name = var.alarm_rule_name

  metric {
    namespace = "SYS.AS"
  }

  resources {
    dimensions {
      name  = "AutoScalingGroup"
      value = huaweicloud_as_group.test.id
    }
  }

  dynamic "condition" {
    for_each = var.rule_conditions

    content {
      alarm_level         = condition.value.alarm_level
      metric_name         = condition.value.metric_name
      period              = condition.value.period
      filter              = condition.value.filter
      comparison_operator = condition.value.comparison_operator
      suppress_duration   = condition.value.suppress_duration
      value               = condition.value.value
      count               = condition.value.count
    }
  }

  alarm_actions {
    type              = "autoscaling"
    notification_list = [huaweicloud_smn_topic.test.id]
  }
}
```

**参数说明**：
- **alarm_name**：告警规则的名称，通过引用输入变量 `alarm_rule_name` 进行赋值
- **metric**：监控指标配置块
  - **namespace**：命名空间，设置为"SYS.AS"表示弹性伸缩服务的监控指标
- **resources**：监控资源配置块
  - **dimensions**：维度配置块，定义监控资源的维度信息
    - **name**：维度名称，设置为"AutoScalingGroup"表示监控AS组
    - **value**：维度值，引用AS组资源（huaweicloud_as_group.test）的ID进行赋值
- **condition**：告警条件配置块，通过动态块（dynamic block）根据输入变量 `rule_conditions` 创建多个告警条件
  - **alarm_level**：告警级别，通过输入变量 `rule_conditions` 中的 `alarm_level` 进行赋值，默认为2（重要）
  - **metric_name**：监控指标名称，通过输入变量 `rule_conditions` 中的 `metric_name` 进行赋值，例如"cpu_util"表示CPU使用率
  - **period**：监控周期（秒），通过输入变量 `rule_conditions` 中的 `period` 进行赋值
  - **filter**：聚合方式，通过输入变量 `rule_conditions` 中的 `filter` 进行赋值，例如"average"表示平均值
  - **comparison_operator**：比较运算符，通过输入变量 `rule_conditions` 中的 `comparison_operator` 进行赋值，例如">"表示大于
  - **suppress_duration**：抑制周期（秒），通过输入变量 `rule_conditions` 中的 `suppress_duration` 进行赋值，默认为0
  - **value**：阈值，通过输入变量 `rule_conditions` 中的 `value` 进行赋值
  - **count**：连续触发次数，通过输入变量 `rule_conditions` 中的 `count` 进行赋值
- **alarm_actions**：告警动作配置块
  - **type**：动作类型，设置为"autoscaling"表示触发弹性伸缩
  - **notification_list**：通知列表，引用SMN主题资源（huaweicloud_smn_topic.test）的ID进行赋值

### 13. 创建AS扩缩容策略资源

在TF文件中添加以下脚本以告知Terraform创建AS扩缩容策略资源：

```hcl
variable "scaling_up_policy_name" {
  description = "扩容策略的名称"
  type        = string
}

variable "scaling_up_cool_down_time" {
  description = "扩容策略的冷却时间"
  type        = number
  default     = 300
}

variable "scaling_up_instance_number" {
  description = "触发扩容策略时添加的实例数量"
  type        = number
  default     = 1
}

variable "scaling_down_policy_name" {
  description = "缩容策略的名称"
  type        = string
}

variable "scaling_down_cool_down_time" {
  description = "缩容策略的冷却时间"
  type        = number
  default     = 300
}

variable "scaling_down_instance_number" {
  description = "触发缩容策略时移除的实例数量"
  type        = number
  default     = 1
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AS扩容策略资源，用于在告警触发时自动扩容
resource "huaweicloud_as_policy" "scaling_up" {
  scaling_policy_name = var.scaling_up_policy_name
  scaling_policy_type = "ALARM"
  scaling_group_id    = huaweicloud_as_group.test.id
  alarm_id            = huaweicloud_ces_alarmrule.test.id
  cool_down_time      = var.scaling_up_cool_down_time

  scaling_policy_action {
    operation       = "ADD"
    instance_number = var.scaling_up_instance_number
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AS缩容策略资源，用于在告警触发时自动缩容
resource "huaweicloud_as_policy" "scaling_down" {
  scaling_policy_name = var.scaling_down_policy_name
  scaling_policy_type = "ALARM"
  scaling_group_id    = huaweicloud_as_group.test.id
  alarm_id            = huaweicloud_ces_alarmrule.test.id
  cool_down_time      = var.scaling_down_cool_down_time

  scaling_policy_action {
    operation       = "REMOVE"
    instance_number = var.scaling_down_instance_number
  }
}
```

**参数说明**（扩容策略）：
- **scaling_policy_name**：扩容策略的名称，通过引用输入变量 `scaling_up_policy_name` 进行赋值
- **scaling_policy_type**：策略类型，设置为"ALARM"表示基于告警的策略
- **scaling_group_id**：AS组ID，引用AS组资源（huaweicloud_as_group.test）的ID进行赋值
- **alarm_id**：告警规则ID，引用CES告警规则资源（huaweicloud_ces_alarmrule.test）的ID进行赋值
- **cool_down_time**：冷却时间（秒），通过引用输入变量 `scaling_up_cool_down_time` 进行赋值，默认为300秒
- **scaling_policy_action**：策略动作配置块
  - **operation**：操作类型，设置为"ADD"表示添加实例
  - **instance_number**：实例数量，通过引用输入变量 `scaling_up_instance_number` 进行赋值，表示每次扩容时添加的实例数量

**参数说明**（缩容策略）：
- **scaling_policy_name**：缩容策略的名称，通过引用输入变量 `scaling_down_policy_name` 进行赋值
- **scaling_policy_type**：策略类型，设置为"ALARM"表示基于告警的策略
- **scaling_group_id**：AS组ID，引用AS组资源（huaweicloud_as_group.test）的ID进行赋值
- **alarm_id**：告警规则ID，引用CES告警规则资源（huaweicloud_ces_alarmrule.test）的ID进行赋值
- **cool_down_time**：冷却时间（秒），通过引用输入变量 `scaling_down_cool_down_time` 进行赋值，默认为300秒
- **scaling_policy_action**：策略动作配置块
  - **operation**：操作类型，设置为"REMOVE"表示移除实例
  - **instance_number**：实例数量，通过引用输入变量 `scaling_down_instance_number` 进行赋值，表示每次缩容时移除的实例数量

### 14. 预设资源部署所需的入参

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
keypair_name        = "tf_test_keypair"
configuration_name  = "tf_test_configuration"

disk_configurations = [
  {
    disk_type   = "SYS"
    volume_type = "SSD"
    volume_size = 40
  }
]

group_name      = "tf_test_group"
topic_name      = "tf_test_topic"
alarm_rule_name = "tf_test_alarm_rule"

rule_conditions = [
  {
    metric_name         = "cpu_util"
    period              = 300
    filter              = "average"
    comparison_operator = ">"
    value               = 80
    count               = 1
  }
]

scaling_up_policy_name   = "tf_test_scaling_up_policy"
scaling_down_policy_name = "tf_test_scaling_down_policy"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 15. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建告警策略
4. 运行 `terraform show` 查看已创建的告警策略

## 参考信息

- [华为云弹性伸缩产品文档](https://support.huaweicloud.com/as/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [AS告警策略最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/as/alarm-policy)
