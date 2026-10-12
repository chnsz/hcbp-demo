# 部署全球加速健康检查

## 应用场景

全球加速（Global Accelerator，GA）是华为云提供的全球网络加速服务，通过将用户流量从最近的接入点引入华为云骨干网并转发至源站，实现就近接入与全球加速。在加速业务中，终端节点组内的源站可能因网络抖动或服务异常而不可用，健康检查能够周期性探测终端节点的可用性，自动隔离异常节点，保障加速链路的稳定性。

本最佳实践将介绍如何使用Terraform自动化部署全球加速健康检查，包括全球加速器创建、监听器配置、终端节点组创建以及健康检查配置，帮助您快速构建具备健康探测能力的全球加速服务。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [全球加速器（huaweicloud_ga_accelerator）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_accelerator)
- [全球加速监听器（huaweicloud_ga_listener）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_listener)
- [全球加速终端节点组（huaweicloud_ga_endpoint_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_endpoint_group)
- [全球加速健康检查（huaweicloud_ga_health_check）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_health_check)

### 资源/数据源依赖关系

```
huaweicloud_ga_accelerator
    └── huaweicloud_ga_listener
            └── huaweicloud_ga_endpoint_group
                    └── huaweicloud_ga_health_check
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建全球加速器

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全球加速器资源
variable "accelerator_name" {
  description = "The name of the GA accelerator"
  type        = string
}

variable "accelerator_description" {
  description = "The description of the GA accelerator"
  type        = string
  default     = "GA accelerator for health check"
}

variable "ip_area" {
  description = "The area of the IP address. Valid values: CM, CT, CU, EU, AP, AF, ME, GE"
  type        = string
  default     = "CM"
}

variable "tags" {
  description = "The tags of the GA accelerator and listener"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_ga_accelerator" "test" {
  name        = var.accelerator_name
  description = var.accelerator_description

  ip_sets {
    ip_type = "IPV4"
    area    = var.ip_area
  }

  ip_sets {
    ip_type = "IPV6"
    area    = var.ip_area
  }

  tags = var.tags
}
```

**参数说明**：
- **name**：全球加速器名称，通过引用输入变量 accelerator_name 进行赋值
- **description**：全球加速器描述，通过引用输入变量 accelerator_description 进行赋值
- **ip_sets.ip_type**：IP地址类型，分别配置为 IPV4 与 IPV6，实现双栈接入
- **ip_sets.area**：IP地址所属区域，通过引用输入变量 ip_area 进行赋值
- **tags**：全球加速器标签，通过引用输入变量 tags 进行赋值

### 3. 创建全球加速监听器

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全球加速监听器资源
variable "listener_name" {
  description = "The name of the GA listener"
  type        = string
}

variable "listener_protocol" {
  description = "The protocol of the GA listener. Valid values: TCP, UDP"
  type        = string
  default     = "TCP"
}

variable "listener_description" {
  description = "The description of the GA listener"
  type        = string
  default     = "GA listener for health check"
}

variable "port_from" {
  description = "The start port of the listener port range"
  type        = number
  default     = 4000
}

variable "port_to" {
  description = "The end port of the listener port range"
  type        = number
  default     = 4200
}

resource "huaweicloud_ga_listener" "test" {
  accelerator_id = huaweicloud_ga_accelerator.test.id
  name           = var.listener_name
  protocol       = var.listener_protocol
  description    = var.listener_description

  port_ranges {
    from_port = var.port_from
    to_port   = var.port_to
  }

  tags = var.tags
}
```

**参数说明**：
- **accelerator_id**：所属全球加速器ID，引用前一步创建的全球加速器资源的ID进行赋值
- **name**：监听器名称，通过引用输入变量 listener_name 进行赋值
- **protocol**：监听器协议，通过引用输入变量 listener_protocol 进行赋值
- **description**：监听器描述，通过引用输入变量 listener_description 进行赋值
- **port_ranges.from_port**：监听端口范围的起始端口，通过引用输入变量 port_from 进行赋值
- **port_ranges.to_port**：监听端口范围的结束端口，通过引用输入变量 port_to 进行赋值
- **tags**：监听器标签，通过引用输入变量 tags 进行赋值

### 4. 创建全球加速终端节点组

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全球加速终端节点组资源
variable "endpoint_group_name" {
  description = "The name of the GA endpoint group"
  type        = string
}

variable "endpoint_group_description" {
  description = "The description of the GA endpoint group"
  type        = string
  default     = "GA endpoint group for health check"
}

variable "backend_region" {
  description = "The region where the backend resources are located"
  type        = string
  default     = "cn-south-1"
}

resource "huaweicloud_ga_endpoint_group" "test" {
  name        = var.endpoint_group_name
  description = var.endpoint_group_description
  region_id   = var.backend_region

  listeners {
    id = huaweicloud_ga_listener.test.id
  }
}
```

**参数说明**：
- **name**：终端节点组名称，通过引用输入变量 endpoint_group_name 进行赋值
- **description**：终端节点组描述，通过引用输入变量 endpoint_group_description 进行赋值
- **region_id**：终端节点组所属后端区域，通过引用输入变量 backend_region 进行赋值
- **listeners.id**：关联的监听器ID，引用前一步创建的监听器资源的ID进行赋值

### 5. 创建全球加速健康检查

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全球加速健康检查资源
variable "health_check_enabled" {
  description = "Whether to enable health check"
  type        = bool
  default     = true
}

variable "health_check_interval" {
  description = "The health check interval in seconds. Range: 1-60"
  type        = number
  default     = 10
}

variable "health_check_max_retries" {
  description = "The maximum number of retries. Range: 1-10"
  type        = number
  default     = 5
}

variable "health_check_port" {
  description = "The port used for health check. Range: 1-65535"
  type        = number
  default     = 8001
}

variable "health_check_timeout" {
  description = "The timeout duration of health check in seconds. Range: 1-60"
  type        = number
  default     = 10
}

resource "huaweicloud_ga_health_check" "test" {
  endpoint_group_id = huaweicloud_ga_endpoint_group.test.id
  enabled           = var.health_check_enabled
  protocol          = "TCP"
  interval          = var.health_check_interval
  max_retries       = var.health_check_max_retries
  port              = var.health_check_port
  timeout           = var.health_check_timeout
}
```

**参数说明**：
- **endpoint_group_id**：所属终端节点组ID，引用前一步创建的终端节点组资源的ID进行赋值
- **enabled**：是否启用健康检查，通过引用输入变量 health_check_enabled 进行赋值
- **protocol**：健康检查协议，固定为 TCP
- **interval**：健康检查间隔时间（秒），通过引用输入变量 health_check_interval 进行赋值
- **max_retries**：健康检查最大重试次数，通过引用输入变量 health_check_max_retries 进行赋值
- **port**：健康检查端口，通过引用输入变量 health_check_port 进行赋值
- **timeout**：健康检查超时时间（秒），通过引用输入变量 health_check_timeout 进行赋值

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 全球加速器参数
accelerator_name    = "ga-accelerator-test"
listener_name       = "ga-listener-test"
endpoint_group_name = "ga-endpoint-group-test"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="accelerator_name=my-accelerator"`
2. 环境变量：`export TF_VAR_accelerator_name=my-accelerator`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建全球加速健康检查
4. 运行 `terraform show` 查看已创建的全球加速健康检查

## 参考信息

- [华为云全球加速产品文档](https://support.huaweicloud.com/ga/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [全球加速健康检查最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ga/ga-health-check)
