# 部署事件订阅（自定义事件源，EG事件目标）

## 应用场景

事件网格（EventGrid，EG）的事件订阅功能是一种基于自定义事件源的事件路由和分发机制，可以实现事件的过滤、转换和分发。通过自定义事件源，您可以构建灵活的事件驱动架构，实现不同系统间的解耦和异步通信。

自定义事件源特别适用于需要自定义事件格式、实现复杂事件过滤、构建事件驱动微服务等场景，如业务系统集成、实时数据处理、事件通知分发等。本最佳实践将介绍如何使用Terraform自动化部署一个完整的自定义事件源到EG事件目标的事件订阅配置，包括自定义事件源、自定义事件通道和事件订阅的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [事件网格连接查询数据源（data.huaweicloud_eg_connections）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/eg_connections)

### 资源

- [自定义事件通道资源（huaweicloud_eg_custom_event_channel）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_custom_event_channel)
- [自定义事件源资源（huaweicloud_eg_custom_event_source）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_custom_event_source)
- [事件订阅资源（huaweicloud_eg_event_subscription）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_event_subscription)
- [时间等待资源（time_sleep）](https://registry.terraform.io/providers/hashicorp/time/latest/docs/resources/sleep)

### 资源/数据源依赖关系

```
data.huaweicloud_eg_connections
    └── huaweicloud_eg_event_subscription

huaweicloud_eg_custom_event_channel
    ├── huaweicloud_eg_custom_event_source
    └── time_sleep
        └── huaweicloud_eg_event_subscription
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建自定义事件通道

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建自定义事件通道资源：

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

### 3. 创建自定义事件源

在TF文件中添加以下脚本以告知Terraform创建自定义事件源资源：

```hcl
# Variable definitions for custom event source
variable "source_name" {
  description = "The name of the custom event source"
  type        = string
}

variable "source_type" {
  description = "The type of the custom event source"
  type        = string
  default     = "APPLICATION"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建自定义事件源资源
resource "huaweicloud_eg_custom_event_source" "test" {
  channel_id = huaweicloud_eg_custom_event_channel.test.id
  name       = var.source_name
  type       = var.source_type
}
```

**参数说明**：
- **channel_id**：事件通道的ID，引用前面创建的自定义事件通道资源的ID
- **name**：自定义事件源的名称，通过引用输入变量source_name进行赋值
- **type**：自定义事件源的类型，通过引用输入变量source_type进行赋值，默认值为"APPLICATION"

### 4. 通过数据源查询事件网格连接信息

在TF文件中添加以下脚本以告知Terraform查询事件网格连接信息：

```hcl
# Variable definitions for connection query
variable "connection_name" {
  description = "The exact name of the connection to be queried"
  type        = string
  default     = "default"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的事件网格连接信息，用于创建事件订阅
data "huaweicloud_eg_connections" "test" {
  name = var.connection_name
}
```

**参数说明**：
- **name**：连接名称，通过引用输入变量connection_name进行赋值，默认值为"default"

### 5. 创建时间等待资源

在TF文件中添加以下脚本以告知Terraform创建时间等待资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建时间等待资源，用于等待自定义事件通道准备就绪
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

### 6. 创建事件订阅资源

在TF文件中添加以下脚本以告知Terraform创建事件订阅资源：

```hcl
# Variable definitions for event subscription
variable "subscription_name" {
  description = "The name of the event subscription"
  type        = string
}

variable "sources_provider_type" {
  description = "The provider type of the event source"
  type        = string
  default     = "CUSTOM"
}

variable "source_op" {
  description = "The operation of the source"
  type        = string
  default     = "StringIn"
}

variable "targets_name" {
  description = "The name of the event target"
  type        = string
  default     = "HTTPS"
}

variable "targets_provider_type" {
  description = "The type of the event target"
  type        = string
  default     = "CUSTOM"
}

variable "transform" {
  description = "The transform configuration of the event target, in JSON format"
  type        = map(string)
  default     = {
    "type" : "ORIGINAL",
  }
}

variable "detail_name" {
  description = "The name(key) of the target detail configuration"
  type        = string
  default     = "detail"
}

variable "target_url" {
  description = "The target url of the event target"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建事件订阅资源
resource "huaweicloud_eg_event_subscription" "test" {
  channel_id = huaweicloud_eg_custom_event_channel.test.id
  name       = var.subscription_name

  sources {
    name          = huaweicloud_eg_custom_event_channel.test.name
    provider_type = var.sources_provider_type

    filter_rule = jsonencode({
      "source" : [
        {
          "op" : var.source_op,
          "values" : [huaweicloud_eg_custom_event_channel.test.name]
        }
      ]
    })
  }

  targets {
    name          = var.targets_name
    provider_type = var.targets_provider_type
    connection_id = try(data.huaweicloud_eg_connections.test.connections[0].id, "")
    transform     = jsonencode(var.transform)
    detail_name   = var.detail_name
    detail        = jsonencode({
      "url" : var.target_url
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
- **channel_id**：事件通道的ID，引用前面创建的自定义事件通道资源的ID
- **name**：事件订阅的名称，通过引用输入变量subscription_name进行赋值
- **sources**：事件源配置块
  - **name**：事件源名称，使用自定义事件通道的名称
  - **provider_type**：事件源提供者类型，通过引用输入变量sources_provider_type进行赋值，默认值为"CUSTOM"
  - **filter_rule**：过滤规则，以JSON格式配置事件源过滤条件
- **targets**：事件目标配置块
  - **name**：事件目标名称，通过引用输入变量targets_name进行赋值，默认值为"HTTPS"
  - **provider_type**：事件目标提供者类型，通过引用输入变量targets_provider_type进行赋值，默认值为"CUSTOM"
  - **connection_id**：连接ID，使用事件网格连接查询数据源的连接ID
  - **transform**：转换配置，以JSON格式配置事件转换规则
  - **detail_name**：目标详情配置的名称，通过引用输入变量detail_name进行赋值，默认值为"detail"
  - **detail**：目标详情配置，以JSON格式配置目标URL等信息
- **lifecycle**：生命周期管理，忽略sources和targets的变更以避免重建订阅
- **depends_on**：显式依赖关系，确保时间等待资源在事件订阅创建前已完成

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 自定义事件通道配置
channel_name = "tf_test_channel"

# 自定义事件源配置
source_name = "tf-test-source"
source_type = "APPLICATION"

# 事件网格连接配置
connection_name = "default"

# 事件订阅配置
subscription_name = "tf-test-subscription"
target_url        = "https://test.com/example"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="channel_name=my-channel" -var="source_name=my-source"`
2. 环境变量：`export TF_VAR_channel_name=my-channel`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建事件订阅（自定义事件源）
4. 运行 `terraform show` 查看已创建的事件订阅（自定义事件源）详情

## 参考信息

- [华为云事件网格产品文档](https://support.huaweicloud.com/eg/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [事件网格事件订阅（自定义事件源，EG事件目标）最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eg/event-subscriptions/custom)
