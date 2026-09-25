# 部署全球加速访问日志

## 应用场景

全球加速（Global Accelerator，GA）是华为云提供的全球网络加速服务，通过在全球部署的接入点就近接入用户流量，并经由华为云骨干网将流量转发至源站，有效降低跨地域访问时延、提升访问体验。访问日志能够记录监听器的访问请求信息，是排查访问异常、分析流量特征以及满足安全审计要求的重要依据。

本最佳实践将介绍如何使用Terraform自动化部署全球加速访问日志，包括创建全球加速器、监听器、LTS日志组与日志流，并将监听器的访问日志投递至LTS进行统一存储与分析。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [全球加速器（huaweicloud_ga_accelerator）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_accelerator)
- [全球加速监听器（huaweicloud_ga_listener）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_listener)
- [LTS日志组（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS日志流（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [全球加速访问日志（huaweicloud_ga_access_log）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ga_access_log)

### 资源/数据源依赖关系

```
huaweicloud_ga_accelerator
    └── huaweicloud_ga_listener
            └── huaweicloud_ga_access_log

huaweicloud_lts_group
    ├── huaweicloud_lts_stream
    │       └── huaweicloud_ga_access_log
    └── huaweicloud_ga_access_log
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建全球加速器

在TF文件（如main.tf）中添加以下脚本以创建全球加速器：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全球加速器资源
variable "accelerator_name" {
  description = "The name of the GA accelerator"
  type        = string
}

variable "accelerator_description" {
  description = "The description of the GA accelerator"
  type        = string
  default     = ""
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
- **name**：通过引用输入变量 accelerator_name 进行赋值，用于指定全球加速器的名称
- **description**：通过引用输入变量 accelerator_description 进行赋值，用于指定全球加速器的描述信息
- **ip_sets.ip_type**：IP地址类型，本实践同时配置 IPV4 与 IPV6 两组IP地址
- **ip_sets.area**：通过引用输入变量 ip_area 进行赋值，用于指定IP地址所属的区域
- **tags**：通过引用输入变量 tags 进行赋值，用于为全球加速器添加标签

### 3. 创建全球加速监听器

在TF文件（如main.tf）中添加以下脚本以创建全球加速监听器：

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
  default     = ""
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
- **accelerator_id**：通过引用全球加速器资源 huaweicloud_ga_accelerator.test 的ID进行赋值，用于关联监听器所属的全球加速器
- **name**：通过引用输入变量 listener_name 进行赋值，用于指定监听器的名称
- **protocol**：通过引用输入变量 listener_protocol 进行赋值，用于指定监听器的协议类型
- **description**：通过引用输入变量 listener_description 进行赋值，用于指定监听器的描述信息
- **port_ranges.from_port**：通过引用输入变量 port_from 进行赋值，用于指定监听端口范围的起始端口
- **port_ranges.to_port**：通过引用输入变量 port_to 进行赋值，用于指定监听端口范围的结束端口
- **tags**：通过引用输入变量 tags 进行赋值，用于为监听器添加标签

### 4. 创建LTS日志组

在TF文件（如main.tf）中添加以下脚本以创建LTS日志组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建LTS日志组资源
variable "lts_group_name" {
  description = "The name of the LTS log group"
  type        = string
}

variable "lts_ttl_in_days" {
  description = "The TTL in days for the LTS log group"
  type        = number
  default     = 30
}

resource "huaweicloud_lts_group" "test" {
  group_name  = var.lts_group_name
  ttl_in_days = var.lts_ttl_in_days
}
```

**参数说明**：
- **group_name**：通过引用输入变量 lts_group_name 进行赋值，用于指定日志组的名称
- **ttl_in_days**：通过引用输入变量 lts_ttl_in_days 进行赋值，用于指定日志的存储时长（单位：天）

### 5. 创建LTS日志流

在TF文件（如main.tf）中添加以下脚本以创建LTS日志流：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建LTS日志流资源
variable "lts_stream_name" {
  description = "The name of the LTS log stream"
  type        = string
}

resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.lts_stream_name
}
```

**参数说明**：
- **group_id**：通过引用LTS日志组资源 huaweicloud_lts_group.test 的ID进行赋值，用于关联日志流所属的日志组
- **stream_name**：通过引用输入变量 lts_stream_name 进行赋值，用于指定日志流的名称

### 6. 创建全球加速访问日志

在TF文件（如main.tf）中添加以下脚本以创建全球加速访问日志：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全球加速访问日志资源
resource "huaweicloud_ga_access_log" "test" {
  resource_type = "LISTENER"
  resource_id   = huaweicloud_ga_listener.test.id
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id
}
```

**参数说明**：
- **resource_type**：日志关联的资源类型，当前仅支持 `LISTENER`
- **resource_id**：通过引用全球加速监听器资源 huaweicloud_ga_listener.test 的ID进行赋值，用于指定需要采集访问日志的监听器
- **log_group_id**：通过引用LTS日志组资源 huaweicloud_lts_group.test 的ID进行赋值，用于指定访问日志投递的目标日志组
- **log_stream_id**：通过引用LTS日志流资源 huaweicloud_lts_stream.test 的ID进行赋值，用于指定访问日志投递的目标日志流

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "<YOUR_ACCESS_KEY>"
secret_key  = "<YOUR_SECRET_KEY>"

# 全球加速器与监听器配置
accelerator_name = "ga-accelerator-test"
listener_name    = "ga-listener-test"

# LTS配置
lts_group_name  = "ga-lts-group"
lts_stream_name = "ga-lts-stream"
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

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建全球加速访问日志
4. 运行 `terraform show` 查看已创建的全球加速访问日志

## 参考信息

- [华为云全球加速产品文档](https://support.huaweicloud.com/ga/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [全球加速访问日志最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ga/ga-access-log)
