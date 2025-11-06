# 部署事件订阅（VPC事件源、EG事件目标）

## 应用场景

华为云事件网格（EventGrid，EG）的事件订阅功能支持将VPC（虚拟私有云）产生的事件自动路由到另一个EG事件通道中，实现事件在不同EG通道间的转发和分发。通过这种集成方式，您可以构建复杂的事件驱动架构，将VPC的网络操作事件（如VPC创建、删除、修改等）实时推送到目标EG通道，供下游消费者进行进一步处理和分析。

本最佳实践特别适用于需要跨区域事件转发、构建事件中继架构、实现VPC事件与EG通道集成的场景，如多区域事件同步、事件审计和监控、复杂事件处理工作流等。本最佳实践将介绍如何使用Terraform自动化部署一个完整的VPC到EG的事件订阅配置，包括VPC网络、自定义事件通道和事件订阅的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [事件网格事件通道查询数据源（data.huaweicloud_eg_event_channels）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/eg_event_channels)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [自定义事件通道资源（huaweicloud_eg_custom_event_channel）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_custom_event_channel)
- [事件订阅资源（huaweicloud_eg_event_subscription）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_event_subscription)
- [时间等待资源（time_sleep）](https://registry.terraform.io/providers/hashicorp/time/latest/docs/resources/sleep)

### 资源/数据源依赖关系

```
data.huaweicloud_eg_event_channels
    └── huaweicloud_eg_event_subscription

huaweicloud_vpc
    └── huaweicloud_vpc_subnet

huaweicloud_eg_custom_event_channel
    ├── time_sleep
    └── huaweicloud_eg_event_subscription

time_sleep.test
    └── huaweicloud_eg_event_subscription
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

### 4. 创建自定义事件通道

在TF文件中添加以下脚本以告知Terraform创建自定义事件通道资源：

```hcl
# Variable definitions for custom event channel
variable "channel_name" {
  description = "The name of the custom event channel"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建自定义事件通道资源
resource "huaweicloud_eg_custom_event_channel" "test" {
  name = var.channel_name
}
```

**参数说明**：
- **name**：自定义事件通道的名称，通过引用输入变量channel_name进行赋值

### 5. 查询事件网格事件通道信息

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

### 6. 创建时间等待资源

在TF文件中添加以下脚本以告知Terraform创建时间等待资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建时间等待资源，用于等待EG通道准备就绪
resource "time_sleep" "test" {
  create_duration = "3s"

  depends_on = [
    huaweicloud_eg_custom_event_channel.test
  ]
}
```

**参数说明**：
- **create_duration**：等待时间，设置为"3s"表示等待3秒
- **depends_on**：显式依赖关系，确保自定义事件通道在时间等待资源创建前已存在

### 7. 创建事件订阅资源

在TF文件中添加以下脚本以告知Terraform创建事件订阅资源：

```hcl
# Variable definitions for event subscription
variable "sources_name" {
  description = "The name of the event source"
  type        = string
  default     = "HC.VPC"
}

variable "sources_provider_type" {
  description = "The provider type of the event source"
  type        = string
  default     = "OFFICIAL"
}

variable "source_op" {
  description = "The operation of the source"
  type        = string
  default     = "StringIn"
}

variable "type_op" {
  description = "The operation of the type"
  type        = string
  default     = "StringIn"
}

variable "subscription_source_values" {
  description = "The event types to be subscribed from VPC service"
  type        = list(string)
  default     = [
    "VPC:CloudTrace:ApiCall",
    "VPC:CloudTrace:ConsoleAction",
    "VPC:CloudTrace:SystemAction"
  ]
}

variable "targets_name" {
  description = "The name of the event target"
  type        = string
  default     = "HC.EG"
}

variable "targets_provider_type" {
  description = "The type of the event target"
  type        = string
  default     = "OFFICIAL"
}

variable "detail_name" {
  description = "The name(key) of the target detail configuration"
  type        = string
  default     = "eg_detail"
}

variable "transform" {
  description = "The transform configuration of the event target, in JSON format"
  type        = map(string)
  default     = {
    "type" : "ORIGINAL",
  }
}

variable "agency_name" {
  description = "The name of the agency"
  type        = string
  default     = "EG_AGENCY"
}

variable "target_project_id" {
  description = "The ID of the target project"
  type        = string
}

variable "target_region_name" {
  description = "The name of the target region"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建事件订阅资源
resource "huaweicloud_eg_event_subscription" "test" {
  channel_id = try(data.huaweicloud_eg_event_channels.test.channels[0].id, "")
  name       = try(data.huaweicloud_eg_event_channels.test.channels[0].name, "")

  sources {
    name          = var.sources_name
    provider_type = var.sources_provider_type

    filter_rule = jsonencode({
      "source" : [
        {
          "op" : var.source_op,
          "values" : ["HC.VPC"]
        }
      ],
      "type" : [
        {
          "op" : var.type_op,
          "values" : var.subscription_source_values
        }
      ],
    })
  }

  targets {
    name          = var.targets_name
    provider_type = var.targets_provider_type
    detail_name   = var.detail_name
    transform     = jsonencode(var.transform)
    detail        = jsonencode({
      "agency_name" : var.agency_name
      "target_channel_id" : huaweicloud_eg_custom_event_channel.test.id
      "target_project_id" : var.target_project_id
      "target_region" : var.target_region_name != "" ? var.target_region_name : var.region_name
    })
  }

  lifecycle {
    ignore_changes = [
      sources, targets
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
  - **name**：事件源名称，通过引用输入变量sources_name进行赋值，默认值为"HC.VPC"
  - **provider_type**：事件源提供者类型，通过引用输入变量sources_provider_type进行赋值，默认值为"OFFICIAL"
  - **filter_rule**：过滤规则，以JSON格式配置事件源过滤条件，包括source和type过滤
- **targets**：事件目标配置块
  - **name**：事件目标名称，通过引用输入变量targets_name进行赋值，默认值为"HC.EG"
  - **provider_type**：事件目标提供者类型，通过引用输入变量targets_provider_type进行赋值，默认值为"OFFICIAL"
  - **detail_name**：目标详情配置的名称，通过引用输入变量detail_name进行赋值，默认值为"eg_detail"
  - **transform**：转换配置，以JSON格式配置事件转换规则，通过引用输入变量transform进行赋值，默认类型为"ORIGINAL"
  - **detail**：目标详情配置，以JSON格式配置目标EG通道信息，包括代理名称、目标通道ID、目标项目ID和目标区域
- **lifecycle.ignore_changes**：生命周期管理，忽略sources和targets的变更以避免重建订阅
- **depends_on**：显式依赖关系，确保时间等待资源在事件订阅创建前已完成

### 8. 预设资源部署所需的入参（可选）

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

# 自定义事件通道配置
channel_name = "tf_test_channel"

# 目标项目配置
target_project_id = "your target project ID"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="channel_name=my-channel"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建事件订阅（VPC事件源、EG事件目标）
4. 运行 `terraform show` 查看已创建的事件订阅（VPC事件源、EG事件目标）详情

## 参考信息

- [华为云事件网格产品文档](https://support.huaweicloud.com/eg/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [事件网格事件订阅（VPC事件源、EG事件目标）最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eg/event-subscriptions/vpc-to-eg)
