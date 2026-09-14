# 部署自定义事件订阅

## 应用场景

事件网格（EventGrid，EG）是华为云提供的事件驱动架构服务，支持事件的生产、路由、转换和消费。通过自定义事件通道与自定义事件源，您可以将业务系统产生的自定义事件接入事件网格，并按需路由到指定的 HTTPS 事件目标。

本最佳实践将介绍如何使用Terraform自动化部署自定义事件通道、自定义事件源以及事件订阅，实现自定义事件从接入、过滤到分发到 HTTPS 目标的完整链路，帮助您快速构建松耦合、可扩展的事件驱动应用。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [EG连接（data.huaweicloud_eg_connections）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/eg_connections)

### 资源

- [EG自定义事件通道（huaweicloud_eg_custom_event_channel）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_custom_event_channel)
- [EG自定义事件源（huaweicloud_eg_custom_event_source）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_custom_event_source)
- [EG事件订阅（huaweicloud_eg_event_subscription）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/eg_event_subscription)
- [延时资源（time_sleep）](https://registry.terraform.io/providers/hashicorp/time/latest/docs/resources/sleep)

### 资源/数据源依赖关系

```
data.huaweicloud_eg_connections
    └── huaweicloud_eg_event_subscription

huaweicloud_eg_custom_event_channel
    ├── huaweicloud_eg_custom_event_source
    ├── time_sleep
    └── huaweicloud_eg_event_subscription

time_sleep
    └── huaweicloud_eg_event_subscription
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建自定义事件通道

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建自定义事件通道
variable "channel_name" {
  description = "The name of the custom event channel"
  type        = string
}

resource "huaweicloud_eg_custom_event_channel" "test" {
  name = var.channel_name
}
```

**参数说明**：
- **name**：自定义事件通道名称，通过引用输入变量 channel_name 进行赋值

### 3. 创建自定义事件源

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建自定义事件源
variable "source_name" {
  description = "The name of the custom event source"
  type        = string
}

variable "source_type" {
  description = "The type of the custom event source"
  type        = string
  default     = "APPLICATION"
}

resource "huaweicloud_eg_custom_event_source" "test" {
  channel_id = huaweicloud_eg_custom_event_channel.test.id
  name       = var.source_name
  type       = var.source_type
}
```

**参数说明**：
- **channel_id**：自定义事件源所属的事件通道ID，引用上一步创建的自定义事件通道的ID进行赋值
- **name**：自定义事件源名称，通过引用输入变量 source_name 进行赋值
- **type**：自定义事件源类型，通过引用输入变量 source_type 进行赋值，默认为 APPLICATION

### 4. 查询EG连接

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询EG连接
variable "connection_name" {
  description = "The exact name of the connection to be queried"
  type        = string
  default     = "default"
}

data "huaweicloud_eg_connections" "test" {
  name = var.connection_name
}
```

**参数说明**：
- **name**：待查询的EG连接名称，通过引用输入变量 connection_name 进行赋值，默认为 default

### 5. 等待自定义事件通道和事件源就绪

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 等待自定义事件通道和事件源就绪
resource "time_sleep" "test" {
  create_duration = "3s"

  depends_on = [
    huaweicloud_eg_custom_event_channel.test
  ]
}
```

**参数说明**：
- **create_duration**：创建延时时间，此处设置为 3s，确保自定义事件通道和事件源就绪后再创建事件订阅
- **depends_on**：显式依赖自定义事件通道资源

### 6. 创建事件订阅

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建事件订阅
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
- **channel_id**：事件订阅所属的事件通道ID，引用自定义事件通道的ID进行赋值
- **name**：事件订阅名称，通过引用输入变量 subscription_name 进行赋值
- **sources.name**：事件源名称，引用自定义事件通道的名称进行赋值
- **sources.provider_type**：事件源提供方类型，通过引用输入变量 sources_provider_type 进行赋值，默认为 CUSTOM
- **sources.filter_rule**：事件源过滤规则，以JSON格式定义，其中 op 通过引用输入变量 source_op 进行赋值（默认为 StringIn），values 引用自定义事件通道的名称
- **targets.name**：事件目标名称，通过引用输入变量 targets_name 进行赋值，默认为 HTTPS
- **targets.provider_type**：事件目标提供方类型，通过引用输入变量 targets_provider_type 进行赋值，默认为 CUSTOM
- **targets.connection_id**：事件目标连接ID，引用查询到的EG连接ID进行赋值
- **targets.transform**：事件目标转换配置，以JSON格式定义，通过引用输入变量 transform 进行赋值，默认为 {"type":"ORIGINAL"}
- **targets.detail_name**：事件目标详情配置的键名，通过引用输入变量 detail_name 进行赋值，默认为 detail
- **targets.detail**：事件目标详情配置，以JSON格式定义，其中 url 通过引用输入变量 target_url 进行赋值
- **lifecycle.ignore_changes**：忽略 sources 和 targets 的变更，避免因服务端默认值导致的差异
- **depends_on**：显式依赖延时资源，确保自定义事件通道和事件源就绪后再创建事件订阅

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
channel_name      = "tf_test_channel"
source_name       = "tf-test-source"
subscription_name = "tf-test-subscription"
target_url        = "https://test.com/example"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="channel_name=my-channel"`
2. 环境变量：`export TF_VAR_channel_name=my-channel`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建自定义事件订阅
4. 运行 `terraform show` 查看已创建的自定义事件订阅

## 参考信息

- [华为云事件网格产品文档](https://support.huaweicloud.com/eg/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EG自定义事件订阅最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eg/event-subscriptions/custom)
