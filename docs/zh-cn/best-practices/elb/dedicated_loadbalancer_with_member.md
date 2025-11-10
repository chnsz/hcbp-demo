# 部署拥有后端实例的专享版弹性负载均衡器

## 应用场景

弹性负载均衡（Elastic Load Balance，ELB）是一种将访问流量自动分发到多台云服务器的服务，能够扩展应用系统对外的服务能力，提高应用的可用性。专享版弹性负载均衡器是华为云提供的高性能、高可用的负载均衡服务，支持TCP、UDP、HTTP、HTTPS等多种协议，适用于需要高性能、高可用性的业务场景。

通过为专享版弹性负载均衡器配置后端服务器组和后端服务器，可以实现流量的智能分发和负载均衡。本最佳实践将介绍如何使用Terraform自动化部署一个拥有后端实例的专享版弹性负载均衡器，包括VPC、子网、负载均衡器、监听器、后端服务器组、健康检查、安全组、安全组规则、ECS实例和后端服务器的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [专享版弹性负载均衡器资源（huaweicloud_elb_loadbalancer）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/elb_loadbalancer)
- [监听器资源（huaweicloud_elb_listener）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/elb_listener)
- [后端服务器组资源（huaweicloud_elb_pool）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/elb_pool)
- [健康检查资源（huaweicloud_elb_monitor）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/elb_monitor)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [后端服务器资源（huaweicloud_elb_member）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/elb_member)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_compute_flavors
    │   └── data.huaweicloud_images_images
    │       └── huaweicloud_compute_instance
    ├── huaweicloud_elb_loadbalancer
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_elb_loadbalancer
        ├── huaweicloud_compute_instance
        └── huaweicloud_elb_member

huaweicloud_elb_loadbalancer
    └── huaweicloud_elb_listener
        └── huaweicloud_elb_pool
            ├── huaweicloud_elb_monitor
            └── huaweicloud_elb_member

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_compute_instance
        └── huaweicloud_elb_member
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询ECS实例资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例：

```hcl
variable "availability_zone" {
  description = "资源所属的可用区名称"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建ECS实例
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询可用区列表，仅当 `var.availability_zone` 为空时查询可用区列表

### 3. 通过数据源查询ECS实例资源创建所需的规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的ECS规格：

```hcl
variable "instance_flavor_id" {
  description = "实例的规格ID"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "实例规格的性能类型"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "实例规格的CPU核心数"
  type        = number
  default     = 2
}

variable "instance_flavor_memory_size" {
  description = "实例规格的内存大小"
  type        = number
  default     = 4
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的ECS规格信息，用于创建ECS实例
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询实例规格信息，仅当 `var.instance_flavor_id` 为空时查询实例规格信息
- **performance_type**：规格的性能类型，通过引用输入变量 `instance_flavor_performance_type` 进行赋值，默认为"normal"表示标准类型
- **cpu_core_count**：CPU核心数，通过引用输入变量 `instance_flavor_cpu_core_count` 进行赋值，默认为2核
- **memory_size**：内存大小（GB），通过引用输入变量 `instance_flavor_memory_size` 进行赋值，默认为4GB
- **availability_zone**：规格所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区

### 4. 通过数据源查询ECS实例资源创建所需的镜像

在TF文件中添加以下脚本以告知Terraform查询符合条件的镜像：

```hcl
variable "instance_image_id" {
  description = "实例的镜像ID"
  type        = string
  default     = ""
}

variable "instance_image_visibility" {
  description = "实例镜像的可见性"
  type        = string
  default     = "public"
}

variable "instance_image_os" {
  description = "实例镜像的操作系统"
  type        = string
  default     = "Ubuntu"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的IMS镜像信息，用于创建ECS实例
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].ids[0], null)
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询镜像信息，仅当 `var.instance_image_id` 为空时查询镜像信息
- **flavor_id**：镜像支持的规格ID，如果指定了实例规格ID则使用该值，否则根据计算规格列表查询数据源的返回结果进行赋值
- **visibility**：镜像的可见性，通过引用输入变量 `instance_image_visibility` 进行赋值，默认为"public"表示公共镜像
- **os**：镜像的操作系统类型，通过引用输入变量 `instance_image_os` 进行赋值，默认为"Ubuntu"操作系统

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
  default     = "172.16.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署弹性负载均衡器
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC的名称，通过引用输入变量 `vpc_name` 进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量 `vpc_cidr` 进行赋值，默认为"172.16.0.0/16"网段

### 6. 创建VPC子网资源

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源（支持IPv6）：

```hcl
variable "subnet_name" {
  description = "子网的名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网的CIDR块"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "子网的网关IP地址"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署弹性负载均衡器
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id      = huaweicloud_vpc.test.id
  name        = var.subnet_name
  cidr        = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip  = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
  ipv6_enable = true
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量 `subnet_name` 进行赋值
- **cidr**：子网的CIDR网段，如果指定了子网CIDR则使用该值，否则使用cidrsubnet函数从VPC的CIDR网段中划分一个子网段
- **gateway_ip**：子网的网关IP，如果指定了网关IP则使用该值，否则使用cidrhost函数从子网网段中获取第一个IP地址作为网关IP
- **ipv6_enable**：是否启用IPv6，设置为true表示启用IPv6功能

### 7. 创建专享版弹性负载均衡器资源

在TF文件中添加以下脚本以告知Terraform创建专享版弹性负载均衡器资源：

```hcl
variable "loadbalancer_name" {
  description = "负载均衡器的名称"
  type        = string
}

variable "loadbalancer_cross_vpc_backend" {
  description = "是否通过后端服务器的IP地址关联后端服务器"
  type        = bool
  default     = false
}

variable "loadbalancer_description" {
  description = "负载均衡器的描述"
  type        = string
  default     = null
}

variable "enterprise_project_id" {
  description = "企业项目ID"
  type        = string
  default     = null
}

variable "loadbalancer_tags" {
  description = "负载均衡器的标签"
  type        = map(string)
  default     = {}
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建专享版弹性负载均衡器资源
resource "huaweicloud_elb_loadbalancer" "test" {
  name                  = var.loadbalancer_name
  vpc_id                = huaweicloud_vpc.test.id
  ipv4_subnet_id        = huaweicloud_vpc_subnet.test.ipv4_subnet_id
  ipv6_network_id       = huaweicloud_vpc_subnet.test.id
  availability_zone     = var.availability_zone != "" ? [var.availability_zone] : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null)
  cross_vpc_backend     = var.loadbalancer_cross_vpc_backend
  description           = var.loadbalancer_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.loadbalancer_tags
}
```

**参数说明**：
- **name**：负载均衡器的名称，通过引用输入变量 `loadbalancer_name` 进行赋值
- **vpc_id**：负载均衡器所属的VPC的ID，引用前面创建的VPC资源的ID
- **ipv4_subnet_id**：负载均衡器的IPv4子网ID，引用子网资源的IPv4子网ID
- **ipv6_network_id**：负载均衡器的IPv6网络ID，引用子网资源的ID（当子网启用IPv6时）
- **availability_zone**：负载均衡器所在的可用区列表，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区
- **cross_vpc_backend**：是否通过后端服务器的IP地址关联后端服务器，通过引用输入变量 `loadbalancer_cross_vpc_backend` 进行赋值，默认为false
- **description**：负载均衡器的描述，通过引用输入变量 `loadbalancer_description` 进行赋值
- **enterprise_project_id**：负载均衡器所属的企业项目ID，通过引用输入变量 `enterprise_project_id` 进行赋值
- **tags**：负载均衡器的标签，通过引用输入变量 `loadbalancer_tags` 进行赋值

### 8. 创建监听器资源

在TF文件中添加以下脚本以告知Terraform创建监听器资源：

```hcl
variable "listener_name" {
  description = "监听器的名称"
  type        = string
}

variable "listener_protocol" {
  description = "监听器的协议"
  type        = string
  default     = "TCP"
}

variable "listener_port" {
  description = "监听器的端口"
  type        = number
  default     = 8080
}

variable "listener_server_certificate" {
  description = "监听器的服务器证书ID，当listener_protocol为HTTPS、TLS或QUIC时必填"
  type        = string
  default     = null
}

variable "listener_ca_certificate" {
  description = "监听器的CA证书ID，仅当listener_protocol为HTTPS时可用"
  type        = string
  default     = null
}

variable "listener_sni_certificates" {
  description = "监听器的SNI证书列表，仅当listener_protocol为HTTPS或TLS时可用"
  type        = list(string)
  default     = []
}

variable "listener_sni_match_algo" {
  description = "监听器的SNI匹配算法"
  type        = string
  default     = null
}

variable "listener_security_policy_id" {
  description = "监听器的安全策略ID，仅当listener_protocol为HTTPS时可用"
  type        = string
  default     = null
}

variable "listener_http2_enable" {
  description = "是否启用HTTP/2，仅当listener_protocol为HTTPS时可用"
  type        = bool
  default     = null
}

variable "listener_port_ranges" {
  description = "监听器的端口范围列表"
  type = list(object({
    start_port = number
    end_port   = number
  }))
  default  = []
  nullable = false
}

variable "listener_idle_timeout" {
  description = "监听器的空闲超时时间"
  type        = number
  default     = 60
}

variable "listener_request_timeout" {
  description = "监听器的请求超时时间"
  type        = number
  default     = null
}

variable "listener_response_timeout" {
  description = "监听器的响应超时时间"
  type        = number
  default     = null
}

variable "listener_description" {
  description = "监听器的描述"
  type        = string
  default     = null
}

variable "listener_tags" {
  description = "监听器的标签"
  type        = map(string)
  default     = {}
}

variable "listener_advanced_forwarding_enabled" {
  description = "是否启用高级转发"
  type        = bool
  default     = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建监听器资源
resource "huaweicloud_elb_listener" "test" {
  loadbalancer_id    = huaweicloud_elb_loadbalancer.test.id
  name               = var.listener_name
  protocol           = var.listener_protocol
  protocol_port      = var.listener_port
  server_certificate = var.listener_server_certificate
  ca_certificate     = var.listener_ca_certificate
  sni_certificate    = var.listener_sni_certificates
  sni_match_algo     = var.listener_sni_match_algo
  security_policy_id = var.listener_security_policy_id
  http2_enable       = var.listener_http2_enable

  dynamic "port_ranges" {
    for_each = var.listener_port_ranges

    content {
      start_port = port_ranges.value["start_port"]
      end_port   = port_ranges.value["end_port"]
    }
  }

  idle_timeout                = var.listener_idle_timeout
  request_timeout             = var.listener_request_timeout
  response_timeout            = var.listener_response_timeout
  description                 = var.listener_description
  tags                        = var.listener_tags
  advanced_forwarding_enabled = var.listener_advanced_forwarding_enabled
}
```

**参数说明**：
- **loadbalancer_id**：监听器所属的负载均衡器ID，引用前面创建的负载均衡器资源的ID
- **name**：监听器的名称，通过引用输入变量 `listener_name` 进行赋值
- **protocol**：监听器的协议，通过引用输入变量 `listener_protocol` 进行赋值，默认为"TCP"
- **protocol_port**：监听器的端口，通过引用输入变量 `listener_port` 进行赋值，默认为8080
- **server_certificate**：监听器的服务器证书ID，当协议为HTTPS、TLS或QUIC时必填，通过引用输入变量 `listener_server_certificate` 进行赋值
- **ca_certificate**：监听器的CA证书ID，仅当协议为HTTPS时可用，通过引用输入变量 `listener_ca_certificate` 进行赋值
- **sni_certificate**：监听器的SNI证书列表，仅当协议为HTTPS或TLS时可用，通过引用输入变量 `listener_sni_certificates` 进行赋值
- **sni_match_algo**：监听器的SNI匹配算法，通过引用输入变量 `listener_sni_match_algo` 进行赋值
- **security_policy_id**：监听器的安全策略ID，仅当协议为HTTPS时可用，通过引用输入变量 `listener_security_policy_id` 进行赋值
- **http2_enable**：是否启用HTTP/2，仅当协议为HTTPS时可用，通过引用输入变量 `listener_http2_enable` 进行赋值
- **port_ranges**：监听器的端口范围列表（动态块），根据输入变量 `listener_port_ranges` 动态创建
  - **start_port**：起始端口，通过引用输入变量中的端口范围配置进行赋值
  - **end_port**：结束端口，通过引用输入变量中的端口范围配置进行赋值
- **idle_timeout**：监听器的空闲超时时间（秒），通过引用输入变量 `listener_idle_timeout` 进行赋值，默认为60秒
- **request_timeout**：监听器的请求超时时间（秒），通过引用输入变量 `listener_request_timeout` 进行赋值
- **response_timeout**：监听器的响应超时时间（秒），通过引用输入变量 `listener_response_timeout` 进行赋值
- **description**：监听器的描述，通过引用输入变量 `listener_description` 进行赋值
- **tags**：监听器的标签，通过引用输入变量 `listener_tags` 进行赋值
- **advanced_forwarding_enabled**：是否启用高级转发，通过引用输入变量 `listener_advanced_forwarding_enabled` 进行赋值，默认为false

### 9. 创建后端服务器组资源

在TF文件中添加以下脚本以告知Terraform创建后端服务器组资源：

```hcl
variable "pool_name" {
  description = "后端服务器组的名称"
  type        = string
  default     = null
}

variable "pool_protocol" {
  description = "后端服务器组的协议"
  type        = string
  default     = "TCP"
}

variable "pool_method" {
  description = "后端服务器组的负载均衡算法"
  type        = string
  default     = "ROUND_ROBIN"
}

variable "pool_any_port_enable" {
  description = "是否启用后端服务器组的任意端口"
  type        = bool
  default     = false
}

variable "pool_description" {
  description = "后端服务器组的描述"
  type        = string
  default     = null
}

variable "pool_persistences" {
  description = "后端服务器组的会话保持配置列表"
  type = list(object({
    type        = string
    cookie_name = optional(string, null)
    timeout     = optional(number, null)
  }))
  default  = []
  nullable = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建后端服务器组资源
resource "huaweicloud_elb_pool" "test" {
  listener_id     = huaweicloud_elb_listener.test.id
  name            = var.pool_name
  protocol        = var.pool_protocol
  lb_method       = var.pool_method
  any_port_enable = var.pool_any_port_enable
  description     = var.pool_description

  dynamic "persistence" {
    for_each = var.pool_persistences

    content {
      type        = persistence.value["type"]
      cookie_name = persistence.value["cookie_name"]
      timeout     = persistence.value["timeout"]
    }
  }
}
```

**参数说明**：
- **listener_id**：后端服务器组所属的监听器ID，引用前面创建的监听器资源的ID
- **name**：后端服务器组的名称，通过引用输入变量 `pool_name` 进行赋值
- **protocol**：后端服务器组的协议，通过引用输入变量 `pool_protocol` 进行赋值，默认为"TCP"
- **lb_method**：后端服务器组的负载均衡算法，通过引用输入变量 `pool_method` 进行赋值，默认为"ROUND_ROBIN"表示轮询算法
- **any_port_enable**：是否启用任意端口，通过引用输入变量 `pool_any_port_enable` 进行赋值，默认为false
- **description**：后端服务器组的描述，通过引用输入变量 `pool_description` 进行赋值
- **persistence**：会话保持配置块（动态块），根据输入变量 `pool_persistences` 动态创建
  - **type**：会话保持类型，通过引用输入变量中的会话保持配置进行赋值
  - **cookie_name**：Cookie名称，通过引用输入变量中的会话保持配置进行赋值
  - **timeout**：会话保持超时时间，通过引用输入变量中的会话保持配置进行赋值

### 10. 创建健康检查资源

在TF文件中添加以下脚本以告知Terraform创建健康检查资源：

```hcl
variable "health_check_protocol" {
  description = "健康检查的协议"
  type        = string
  default     = "TCP"
}

variable "health_check_interval" {
  description = "健康检查的间隔时间"
  type        = number
  default     = 20
}

variable "health_check_timeout" {
  description = "健康检查的超时时间"
  type        = number
  default     = 15
}

variable "health_check_max_retries" {
  description = "健康检查的最大重试次数"
  type        = number
  default     = 10
}

variable "health_check_port" {
  description = "健康检查的端口"
  type        = number
  default     = null
}

variable "health_check_url_path" {
  description = "健康检查的URL路径"
  type        = string
  default     = null
}

variable "health_check_status_code" {
  description = "健康检查的状态码"
  type        = string
  default     = null
}

variable "health_check_http_method" {
  description = "健康检查的HTTP方法"
  type        = string
  default     = null
}

variable "health_check_domain_name" {
  description = "健康检查的域名"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建健康检查资源
resource "huaweicloud_elb_monitor" "test" {
  pool_id     = huaweicloud_elb_pool.test.id
  protocol    = var.health_check_protocol
  interval    = var.health_check_interval
  timeout     = var.health_check_timeout
  max_retries = var.health_check_max_retries
  port        = var.health_check_port
  url_path    = var.health_check_url_path
  status_code = var.health_check_status_code
  http_method = var.health_check_http_method
  domain_name = var.health_check_domain_name
}
```

**参数说明**：
- **pool_id**：健康检查所属的后端服务器组ID，引用前面创建的后端服务器组资源的ID
- **protocol**：健康检查的协议，通过引用输入变量 `health_check_protocol` 进行赋值，默认为"TCP"
- **interval**：健康检查的间隔时间（秒），通过引用输入变量 `health_check_interval` 进行赋值，默认为20秒
- **timeout**：健康检查的超时时间（秒），通过引用输入变量 `health_check_timeout` 进行赋值，默认为15秒
- **max_retries**：健康检查的最大重试次数，通过引用输入变量 `health_check_max_retries` 进行赋值，默认为10次
- **port**：健康检查的端口，通过引用输入变量 `health_check_port` 进行赋值
- **url_path**：健康检查的URL路径，通过引用输入变量 `health_check_url_path` 进行赋值
- **status_code**：健康检查的状态码，通过引用输入变量 `health_check_status_code` 进行赋值
- **http_method**：健康检查的HTTP方法，通过引用输入变量 `health_check_http_method` 进行赋值
- **domain_name**：健康检查的域名，通过引用输入变量 `health_check_domain_name` 进行赋值

### 11. 创建安全组资源

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "安全组的名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署ECS实例
resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**参数说明**：
- **name**：安全组的名称，通过引用输入变量 `security_group_name` 进行赋值

### 12. 创建安全组规则资源

在TF文件中添加以下脚本以告知Terraform创建安全组规则资源：

```hcl
variable "member_protocol_port" {
  description = "后端服务器的端口"
  type        = number
  default     = 8080
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组规则资源
resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  ethertype         = "IPv4"
  direction         = "ingress"
  protocol          = var.pool_protocol == "UDP" ? "udp" : "tcp"
  # 后端服务器端口和健康检查端口
  ports             = var.health_check_port != null ? join(",", distinct([var.health_check_port, var.member_protocol_port])) : var.member_protocol_port
  # ELB后端子网所属的CIDR
  remote_ip_prefix  = huaweicloud_vpc_subnet.test.cidr
}

# 当协议为UDP时，需要创建IPv4 ICMP规则
resource "huaweicloud_networking_secgroup_rule" "in_v4_icmp" {
  count = var.pool_protocol == "UDP" ? 1 : 0

  security_group_id = huaweicloud_networking_secgroup.test.id
  ethertype         = "IPv4"
  direction         = "ingress"
  protocol          = "icmp"
  remote_ip_prefix  = huaweicloud_vpc_subnet.test.cidr
}

# 当协议为UDP时，需要创建IPv6 ICMP规则
resource "huaweicloud_networking_secgroup_rule" "in_v6_icmp" {
  count = var.pool_protocol == "UDP" ? 1 : 0

  security_group_id = huaweicloud_networking_secgroup.test.id
  ethertype         = "IPv6"
  direction         = "ingress"
  protocol          = "icmp"
  remote_ip_prefix  = huaweicloud_vpc_subnet.test.ipv6_cidr
}

# 当协议为UDP时，需要创建IPv6 UDP规则
resource "huaweicloud_networking_secgroup_rule" "in_v6" {
  count = var.pool_protocol == "UDP" ? 1 : 0

  security_group_id = huaweicloud_networking_secgroup.test.id
  ethertype         = "IPv6"
  direction         = "ingress"
  protocol          = "udp"
  # 后端服务器端口和健康检查端口
  ports             = var.health_check_port != null ? join(",", distinct([var.health_check_port, var.member_protocol_port])) : var.member_protocol_port
  # ELB后端子网所属的CIDR
  remote_ip_prefix  = huaweicloud_vpc_subnet.test.ipv6_cidr
}
```

**参数说明**：
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **ethertype**：以太网类型，设置为"IPv4"或"IPv6"
- **direction**：方向，设置为"ingress"表示入站规则
- **protocol**：协议类型，根据后端服务器组协议动态设置，如果协议为UDP则使用"udp"，否则使用"tcp"
- **ports**：端口范围，如果指定了健康检查端口则包含健康检查端口和后端服务器端口，否则仅包含后端服务器端口
- **remote_ip_prefix**：远程IP前缀，设置为子网的CIDR网段，允许来自ELB后端子网的流量
- **count**：资源的创建数，用于控制是否创建IPv4 ICMP、IPv6 ICMP和IPv6 UDP规则，仅当协议为UDP时创建这些规则

> 注意：当后端服务器组协议为UDP时，需要创建IPv4 ICMP、IPv6 ICMP和IPv6 UDP规则，以支持UDP协议的健康检查和流量转发。

### 13. 创建ECS实例资源

在TF文件中添加以下脚本以告知Terraform创建ECS实例资源：

```hcl
variable "instance_name" {
  description = "ECS实例的名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  name              = var.instance_name
  image_id          = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
  flavor_id         = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  security_groups   = [huaweicloud_networking_secgroup.test.name]

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  lifecycle {
    ignore_changes = [
      flavor_id,
      image_id,
      availability_zone
    ]
  }
}
```

**参数说明**：
- **name**：ECS实例的名称，通过引用输入变量 `instance_name` 进行赋值
- **image_id**：ECS实例所使用镜像的ID，如果指定了镜像ID则使用该值，否则根据镜像列表查询数据源的返回结果进行赋值
- **flavor_id**：ECS实例所使用规格的ID，如果指定了实例规格ID则使用该值，否则根据计算规格列表查询数据源的返回结果进行赋值
- **availability_zone**：ECS实例所在的可用区，如果指定了可用区则使用该值，否则使用可用区列表查询数据源的第一个可用区
- **security_groups**：与ECS实例关联的安全组名称列表，使用创建的安全组资源的名称
- **network**：网络配置块，指定ECS实例连接的网络
  - **uuid**：网络的唯一标识符，使用前面创建的子网资源的ID
- **lifecycle**：生命周期配置块，用于控制资源的生命周期行为
  - **ignore_changes**：指定在后续应用中要忽略的属性变更，设置为忽略 `flavor_id`、`image_id` 和 `availability_zone` 的变更

### 14. 创建后端服务器资源

在TF文件中添加以下脚本以告知Terraform创建后端服务器资源：

```hcl
variable "member_weight" {
  description = "后端服务器的权重"
  type        = number
  default     = 1
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建后端服务器资源
resource "huaweicloud_elb_member" "test" {
  pool_id       = huaweicloud_elb_pool.test.id
  address       = huaweicloud_compute_instance.test.access_ip_v4
  protocol_port  = var.member_protocol_port
  weight        = var.member_weight
  subnet_id     = huaweicloud_vpc_subnet.test.ipv4_subnet_id
}
```

**参数说明**：
- **pool_id**：后端服务器所属的后端服务器组ID，引用前面创建的后端服务器组资源的ID
- **address**：后端服务器的IP地址，引用ECS实例资源的IPv4访问地址
- **protocol_port**：后端服务器的端口，通过引用输入变量 `member_protocol_port` 进行赋值，默认为8080
- **weight**：后端服务器的权重，通过引用输入变量 `member_weight` 进行赋值，默认为1
- **subnet_id**：后端服务器所在的子网ID，引用子网资源的IPv4子网ID

> 注意：后端服务器的IP地址必须与负载均衡器在同一VPC内，且后端服务器的端口必须与监听器的端口一致。

### 15. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 网络资源配置
vpc_name                       = "tf_test_vpc"
subnet_name                    = "tf_test_subnet"
security_group_name            = "tf_test_security_group"

# ECS实例配置
instance_name                  = "tf_test_instance"

# 负载均衡器配置
loadbalancer_name              = "tf_test_dedicated_loadbalancer"
loadbalancer_cross_vpc_backend = true

# 监听器配置
listener_name                  = "tf_test_dedicated_listener"

# 后端服务器组配置
pool_name                      = "tf_test_dedicated_pool"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="loadbalancer_name=my-loadbalancer"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 16. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建拥有后端实例的专享版弹性负载均衡器
4. 运行 `terraform show` 查看已创建的拥有后端实例的专享版弹性负载均衡器详情

## 参考信息

- [华为云弹性负载均衡产品文档](https://support.huaweicloud.com/elb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ELB拥有后端实例的专享版弹性负载均衡器最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/elb/dedicated-loadbalancer-with-full-configuration)
