# 部署专用负载均衡器与弹性伸缩

## 应用场景

弹性负载均衡（Elastic Load Balance，ELB）是一种将访问流量自动分发到多台云服务器的服务，能够扩展应用系统对外的服务能力，提高应用的可用性。弹性伸缩（Auto Scaling，AS）是一种根据业务需求和策略自动调整计算资源的服务，能够根据业务负载自动增加或减少云服务器数量，实现资源的弹性伸缩。

通过将专用负载均衡器与弹性伸缩服务结合使用，可以实现自动化的流量分发和资源弹性伸缩，当业务负载增加时，弹性伸缩服务会自动增加云服务器实例，并将新增实例自动添加到负载均衡器的后端服务器组中；当业务负载减少时，弹性伸缩服务会自动减少云服务器实例，实现资源的自动优化和成本控制。本最佳实践将介绍如何使用Terraform自动化部署专用负载均衡器与弹性伸缩的集成方案，包括VPC、子网、专用负载均衡器、监听器、后端服务器组、弹性伸缩配置、弹性伸缩组、告警规则和伸缩策略的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区数据源（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ELB规格数据源（huaweicloud_elb_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/elb_flavors)
- [ECS规格数据源（huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像数据源（huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [专用负载均衡器资源（huaweicloud_elb_loadbalancer）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/elb_loadbalancer)
- [EIP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [EIP绑定资源（huaweicloud_vpc_eipv3_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eipv3_associate)
- [监听器资源（huaweicloud_elb_listener）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/elb_listener)
- [后端服务器组资源（huaweicloud_elb_pool）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/elb_pool)
- [弹性伸缩配置资源（huaweicloud_as_configuration）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_configuration)
- [弹性伸缩组资源（huaweicloud_as_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_group)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [实例附加到伸缩组资源（huaweicloud_as_instance_attach）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_instance_attach)
- [告警规则资源（huaweicloud_ces_alarmrule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarmrule)
- [伸缩策略资源（huaweicloud_as_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_policy)

### 资源/数据源依赖关系

```text
data.huaweicloud_availability_zones
    ├── data.huaweicloud_elb_flavors
    │   └── huaweicloud_elb_loadbalancer
    ├── data.huaweicloud_compute_flavors
    │   └── data.huaweicloud_images_images
    │       ├── huaweicloud_as_configuration
    │       └── huaweicloud_compute_instance
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_elb_loadbalancer
        ├── huaweicloud_as_group
        └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_as_configuration
    └── huaweicloud_compute_instance

huaweicloud_elb_loadbalancer
    ├── huaweicloud_vpc_eipv3_associate
    └── huaweicloud_elb_listener
        └── huaweicloud_elb_pool
            └── huaweicloud_as_group

huaweicloud_vpc_eip
    └── huaweicloud_vpc_eipv3_associate

huaweicloud_as_configuration
    └── huaweicloud_as_group
        ├── huaweicloud_as_instance_attach
        └── huaweicloud_ces_alarmrule
            └── huaweicloud_as_policy
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询数据源

在TF文件（如main.tf）中添加以下脚本以查询可用区、ELB规格、ECS规格和镜像信息：

```hcl
# 查询可用区信息
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

# 查询ELB规格信息
data "huaweicloud_elb_flavors" "test" {
  type = "L7"
}

# 查询ECS规格信息（仅在未指定flavor_id时查询）
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}

# 查询镜像信息（仅在未指定image_id时查询）
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].ids[0], null)
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}
```

**参数说明**：
- **type**：ELB规格类型，设置为"L7"（七层负载均衡）
- **performance_type**：ECS规格性能类型，通过引用输入变量instance_flavor_performance_type进行赋值，默认值为"normal"
- **cpu_core_count**：CPU核数，通过引用输入变量instance_flavor_cpu_core_count进行赋值，默认值为2
- **memory_size**：内存大小，通过引用输入变量instance_flavor_memory_size进行赋值，默认值为4（GB）
- **availability_zone**：可用区，通过引用输入变量或可用区数据源进行赋值
- **flavor_id**：规格ID，通过引用输入变量或ECS规格数据源进行赋值
- **visibility**：镜像可见性，通过引用输入变量instance_image_visibility进行赋值，默认值为"public"
- **os**：操作系统，通过引用输入变量instance_image_os进行赋值，默认值为"Ubuntu"

### 3. 创建基础网络资源

在TF文件（如main.tf）中添加以下脚本以创建VPC、子网和安全组：

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "172.16.0.0/16"
}

variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
}

variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 创建VPC
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

# 创建子网
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}

# 创建安全组
resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

### 4. 创建专用负载均衡器资源

在TF文件（如main.tf）中添加以下脚本以创建专用负载均衡器：

```hcl
variable "loadbalancer_name" {
  description = "The name of the loadbalancer"
  type        = string
}

variable "loadbalancer_cross_vpc_backend" {
  description = "Whether to associate backend servers with the load balancer by using their IP addresses"
  type        = bool
  default     = false
}

variable "loadbalancer_description" {
  description = "The description of the loadbalancer"
  type        = string
  default     = null
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = null
}

variable "loadbalancer_tags" {
  description = "The tags of the loadbalancer"
  type        = map(string)
  default     = {}
}

variable "loadbalancer_force_delete" {
  description = "Whether to force delete the loadbalancer"
  type        = bool
  default     = true
}

# 创建专用负载均衡器
resource "huaweicloud_elb_loadbalancer" "test" {
  name                  = var.loadbalancer_name
  vpc_id                = huaweicloud_vpc.test.id
  ipv4_subnet_id        = huaweicloud_vpc_subnet.test.ipv4_subnet_id
  l7_flavor_id          = try(data.huaweicloud_elb_flavors.test.ids[0], null)
  availability_zone     = var.availability_zone != "" ? [var.availability_zone] : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null)
  cross_vpc_backend     = var.loadbalancer_cross_vpc_backend
  description           = var.loadbalancer_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.loadbalancer_tags
  force_delete          = var.loadbalancer_force_delete
}
```

**参数说明**：
- **name**：负载均衡器名称，通过引用输入变量loadbalancer_name进行赋值
- **vpc_id**：VPC ID，通过引用VPC资源进行赋值
- **ipv4_subnet_id**：IPv4子网ID，通过引用子网资源进行赋值
- **l7_flavor_id**：L7规格ID，通过引用ELB规格数据源进行赋值
- **availability_zone**：可用区列表，通过引用输入变量或可用区数据源进行赋值
- **cross_vpc_backend**：是否开启跨VPC后端，通过引用输入变量loadbalancer_cross_vpc_backend进行赋值，默认值为false
- **description**：负载均衡器描述，通过引用输入变量loadbalancer_description进行赋值，可选参数
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，可选参数
- **tags**：标签，通过引用输入变量loadbalancer_tags进行赋值，可选参数
- **force_delete**：是否强制删除，通过引用输入变量loadbalancer_force_delete进行赋值，默认值为true

### 5. 创建EIP资源并绑定到负载均衡器

在TF文件（如main.tf）中添加以下脚本以创建EIP并绑定到负载均衡器：

```hcl
variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "bandwidth_name" {
  description = "The name of the EIP bandwidth"
  type        = string
  default     = "tf_test_eip"
}

variable "bandwidth_size" {
  description = "The bandwidth size of the EIP in Mbps"
  type        = number
  default     = 5
}

variable "bandwidth_share_type" {
  description = "The share type of the bandwidth"
  type        = string
  default     = "PER"
}

variable "bandwidth_charge_mode" {
  description = "The charge mode of the bandwidth"
  type        = string
  default     = "traffic"
}

# 创建EIP
resource "huaweicloud_vpc_eip" "test" {
  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.bandwidth_name
    size        = var.bandwidth_size
    share_type  = var.bandwidth_share_type
    charge_mode = var.bandwidth_charge_mode
  }
}

# 将EIP绑定到负载均衡器
resource "huaweicloud_vpc_eipv3_associate" "test" {
  publicip_id             = huaweicloud_vpc_eip.test.id
  associate_instance_type = "ELB"
  associate_instance_id   = huaweicloud_elb_loadbalancer.test.id
}
```

**参数说明**：
- **publicip.type**：EIP类型，通过引用输入变量eip_type进行赋值，默认值为"5_bgp"
- **bandwidth.name**：带宽名称，通过引用输入变量bandwidth_name进行赋值，默认值为"tf_test_eip"
- **bandwidth.size**：带宽大小，通过引用输入变量bandwidth_size进行赋值，默认值为5（Mbps）
- **bandwidth.share_type**：带宽共享类型，通过引用输入变量bandwidth_share_type进行赋值，默认值为"PER"
- **bandwidth.charge_mode**：带宽计费模式，通过引用输入变量bandwidth_charge_mode进行赋值，默认值为"traffic"
- **associate_instance_type**：关联实例类型，设置为"ELB"
- **associate_instance_id**：关联实例ID，通过引用负载均衡器资源进行赋值

### 6. 创建监听器资源

在TF文件（如main.tf）中添加以下脚本以创建监听器：

```hcl
variable "listener_name" {
  description = "The name of the listener"
  type        = string
}

variable "listener_protocol" {
  description = "The protocol of the listener"
  type        = string
  default     = "HTTP"
}

variable "listener_port" {
  description = "The port of the listener"
  type        = number
  default     = 8080
}

variable "listener_idle_timeout" {
  description = "The idle timeout of the listener"
  type        = number
  default     = 60
}

variable "listener_request_timeout" {
  description = "The request timeout of the listener"
  type        = number
  default     = null
}

variable "listener_response_timeout" {
  description = "The response timeout of the listener"
  type        = number
  default     = null
}

variable "listener_description" {
  description = "The description of the listener"
  type        = string
  default     = null
}

variable "listener_tags" {
  description = "The tags of the listener"
  type        = map(string)
  default     = {}
}

# 创建监听器
resource "huaweicloud_elb_listener" "test" {
  loadbalancer_id  = huaweicloud_elb_loadbalancer.test.id
  name             = var.listener_name
  protocol         = var.listener_protocol
  protocol_port    = var.listener_port
  idle_timeout     = var.listener_idle_timeout
  request_timeout  = var.listener_request_timeout
  response_timeout = var.listener_response_timeout
  description      = var.listener_description
  tags             = var.listener_tags
}
```

**参数说明**：
- **loadbalancer_id**：负载均衡器ID，通过引用负载均衡器资源进行赋值
- **name**：监听器名称，通过引用输入变量listener_name进行赋值
- **protocol**：协议类型，通过引用输入变量listener_protocol进行赋值，默认值为"HTTP"
- **protocol_port**：监听端口，通过引用输入变量listener_port进行赋值，默认值为8080
- **idle_timeout**：空闲超时时间，通过引用输入变量listener_idle_timeout进行赋值，默认值为60（秒）
- **request_timeout**：请求超时时间，通过引用输入变量listener_request_timeout进行赋值，可选参数
- **response_timeout**：响应超时时间，通过引用输入变量listener_response_timeout进行赋值，可选参数
- **description**：监听器描述，通过引用输入变量listener_description进行赋值，可选参数
- **tags**：标签，通过引用输入变量listener_tags进行赋值，可选参数

### 7. 创建后端服务器组资源

在TF文件（如main.tf）中添加以下脚本以创建后端服务器组：

```hcl
variable "pool_name" {
  description = "The name of the pool"
  type        = string
  default     = null
}

variable "pool_protocol" {
  description = "The protocol of the pool"
  type        = string
  default     = "HTTP"
}

variable "pool_method" {
  description = "The load balancing method of the pool"
  type        = string
  default     = "ROUND_ROBIN"
}

variable "pool_any_port_enable" {
  description = "Whether to enable any port for the pool"
  type        = bool
  default     = false
}

variable "pool_description" {
  description = "The description of the pool"
  type        = string
  default     = null
}

variable "pool_persistences" {
  description = "The persistence configurations for the pool"

  type = list(object({
    type        = string
    cookie_name = optional(string, null)
    timeout     = optional(number, null)
  }))

  default  = []
  nullable = false
}

# 创建后端服务器组
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
- **listener_id**：监听器ID，通过引用监听器资源进行赋值
- **name**：后端服务器组名称，通过引用输入变量pool_name进行赋值，可选参数
- **protocol**：协议类型，通过引用输入变量pool_protocol进行赋值，默认值为"HTTP"
- **lb_method**：负载均衡算法，通过引用输入变量pool_method进行赋值，默认值为"ROUND_ROBIN"（轮询）
- **any_port_enable**：是否启用任意端口，通过引用输入变量pool_any_port_enable进行赋值，默认值为false
- **description**：后端服务器组描述，通过引用输入变量pool_description进行赋值，可选参数
- **persistence**：会话保持配置，通过动态块进行配置，可选参数

### 8. 创建弹性伸缩配置资源

在TF文件（如main.tf）中添加以下脚本以创建弹性伸缩配置：

```hcl
variable "configuration_name" {
  description = "The name of the scaling configuration"
  type        = string
}

variable "configuration_image_id" {
  description = "The image ID of the scaling configuration"
  type        = string
  default     = ""
}

variable "configuration_flavor_id" {
  description = "The flavor ID of the scaling configuration"
  type        = string
  default     = ""
}

variable "configuration_user_data" {
  description = "The user data for the scaling configuration instances"
  type        = string
}

variable "configuration_disks" {
  description = "The disk configurations for the scaling configuration instances"

  type = list(object({
    size        = number
    volume_type = string
    disk_type   = string
  }))

  nullable = false
}

# 创建弹性伸缩配置
resource "huaweicloud_as_configuration" "test" {
  scaling_configuration_name = var.configuration_name

  instance_config {
    image              = var.configuration_image_id != "" ? var.configuration_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
    flavor             = var.configuration_flavor_id != "" ? var.configuration_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
    security_group_ids = [huaweicloud_networking_secgroup.test.id]
    user_data          = var.configuration_user_data

    dynamic "disk" {
      for_each = var.configuration_disks

      content {
        size        = disk.value["size"]
        volume_type = disk.value["volume_type"]
        disk_type   = disk.value["disk_type"]
      }
    }
  }
}
```

**参数说明**：
- **scaling_configuration_name**：伸缩配置名称，通过引用输入变量configuration_name进行赋值
- **instance_config.image**：镜像ID，通过引用输入变量或镜像数据源进行赋值
- **instance_config.flavor**：规格ID，通过引用输入变量或ECS规格数据源进行赋值
- **instance_config.security_group_ids**：安全组ID列表，通过引用安全组资源进行赋值
- **instance_config.user_data**：用户数据，通过引用输入变量configuration_user_data进行赋值，用于实例初始化脚本
- **instance_config.disk**：磁盘配置，通过动态块进行配置，支持系统盘和数据盘

### 9. 创建弹性伸缩组资源

在TF文件（如main.tf）中添加以下脚本以创建弹性伸缩组，并将伸缩组与负载均衡器关联：

```hcl
variable "group_name" {
  description = "The name of the AS group"
  type        = string
}

variable "group_desire_instance_number" {
  description = "The desire instance number of the AS group"
  type        = number
  default     = 0
}

variable "group_min_instance_number" {
  description = "The min instance number of the AS group"
  type        = number
  default     = 0
}

variable "group_max_instance_number" {
  description = "The max instance number of the AS group"
  type        = number
  default     = 10
}

variable "group_delete_publicip" {
  description = "Whether to delete the public IP address of the AS group"
  type        = bool
  default     = true
}

variable "group_delete_instances" {
  description = "Whether to delete the instances of the AS group"
  type        = bool
  default     = true
}

variable "group_force_delete" {
  description = "Whether to force delete the AS group"
  type        = bool
  default     = true
}

# 创建弹性伸缩组
resource "huaweicloud_as_group" "test" {
  scaling_group_name       = var.group_name
  scaling_configuration_id = huaweicloud_as_configuration.test.id
  desire_instance_number   = var.group_desire_instance_number
  min_instance_number      = var.group_min_instance_number
  max_instance_number      = var.group_max_instance_number
  vpc_id                   = huaweicloud_vpc.test.id
  delete_publicip          = var.group_delete_publicip
  delete_instances         = var.group_delete_instances
  force_delete             = var.group_force_delete

  networks {
    id = huaweicloud_vpc_subnet.test.id
  }

  lbaas_listeners {
    pool_id       = huaweicloud_elb_pool.test.id
    protocol_port = huaweicloud_elb_listener.test.protocol_port
  }

  lifecycle {
    ignore_changes = [
      # When instances are auto-scaled, the desire instance number will be changed.
      desire_instance_number,
    ]
  }
}
```

**参数说明**：
- **scaling_group_name**：伸缩组名称，通过引用输入变量group_name进行赋值
- **scaling_configuration_id**：伸缩配置ID，通过引用弹性伸缩配置资源进行赋值
- **desire_instance_number**：期望实例数，通过引用输入变量group_desire_instance_number进行赋值，默认值为0
- **min_instance_number**：最小实例数，通过引用输入变量group_min_instance_number进行赋值，默认值为0
- **max_instance_number**：最大实例数，通过引用输入变量group_max_instance_number进行赋值，默认值为10
- **vpc_id**：VPC ID，通过引用VPC资源进行赋值
- **delete_publicip**：删除时是否删除公网IP，通过引用输入变量group_delete_publicip进行赋值，默认值为true
- **delete_instances**：删除时是否删除实例，通过引用输入变量group_delete_instances进行赋值，默认值为true
- **force_delete**：是否强制删除，通过引用输入变量group_force_delete进行赋值，默认值为true
- **networks.id**：网络子网ID，通过引用子网资源进行赋值
- **lbaas_listeners.pool_id**：后端服务器组ID，通过引用后端服务器组资源进行赋值
- **lbaas_listeners.protocol_port**：监听端口，通过引用监听器资源进行赋值

> 注意：通过`lbaas_listeners`配置，弹性伸缩组会自动将新创建的实例添加到指定的负载均衡器后端服务器组中。通过`lifecycle.ignore_changes`可以避免Terraform在后续更新时修改`desire_instance_number`，因为该值会随着弹性伸缩自动变化。

### 10. 创建ECS实例并附加到伸缩组

在TF文件（如main.tf）中添加以下脚本以创建ECS实例并附加到弹性伸缩组：

```hcl
variable "instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "instance_flavor_id" {
  description = "The flavor ID of the instance"
  type        = string
  default     = ""
}

variable "instance_image_id" {
  description = "The image ID of the instance"
  type        = string
  default     = ""
}

variable "availability_zone" {
  description = "The name of the availability zone to which the resources belong"
  type        = string
  default     = ""
}

# 创建ECS实例
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
      availability_zone,
    ]
  }
}

# 将ECS实例附加到弹性伸缩组
resource "huaweicloud_as_instance_attach" "test" {
  scaling_group_id = huaweicloud_as_group.test.id
  instance_id      = huaweicloud_compute_instance.test.id
}
```

**参数说明**：
- **name**：ECS实例名称，通过引用输入变量instance_name进行赋值
- **image_id**：镜像ID，通过引用输入变量或镜像数据源进行赋值
- **flavor_id**：规格ID，通过引用输入变量或ECS规格数据源进行赋值
- **availability_zone**：可用区，通过引用输入变量或可用区数据源进行赋值
- **security_groups**：安全组列表，通过引用安全组资源进行赋值
- **network.uuid**：网络子网ID，通过引用子网资源进行赋值
- **scaling_group_id**：伸缩组ID，通过引用弹性伸缩组资源进行赋值
- **instance_id**：实例ID，通过引用ECS实例资源进行赋值

### 11. 创建告警规则资源

在TF文件（如main.tf）中添加以下脚本以创建告警规则：

```hcl
variable "alarm_rule_name" {
  description = "The name of the CES alarm rule"
  type        = string
}

variable "alarm_rule_conditions" {
  description = "The conditions of the CES alarm rule"

  type = list(object({
    period              = number
    filter              = string
    comparison_operator = string
    value               = number
    unit                = string
    count               = number
    alarm_level         = number
    metric_name         = string
  }))

  nullable = false
}

# 创建告警规则
resource "huaweicloud_ces_alarmrule" "test" {
  alarm_name = var.alarm_rule_name

  metric {
    namespace = "SYS.AS"

    dimensions {
      name  = "AutoScalingGroup"
      value = huaweicloud_as_group.test.id
    }
  }

  dynamic "condition" {
    for_each = var.alarm_rule_conditions

    content {
      period              = condition.value["period"]
      filter              = condition.value["filter"]
      comparison_operator = condition.value["comparison_operator"]
      value               = condition.value["value"]
      unit                = condition.value["unit"]
      count               = condition.value["count"]
      alarm_level         = condition.value["alarm_level"]
      metric_name         = condition.value["metric_name"]
    }
  }

  alarm_actions {
    type              = "autoscaling"
    notification_list = []
  }
}
```

**参数说明**：
- **alarm_name**：告警规则名称，通过引用输入变量alarm_rule_name进行赋值
- **metric.namespace**：命名空间，设置为"SYS.AS"（弹性伸缩服务）
- **metric.dimensions.name**：维度名称，设置为"AutoScalingGroup"
- **metric.dimensions.value**：维度值，通过引用弹性伸缩组资源进行赋值
- **condition.period**：统计周期，通过引用输入变量进行赋值
- **condition.filter**：过滤条件，通过引用输入变量进行赋值，如"max"（最大值）
- **condition.comparison_operator**：比较运算符，通过引用输入变量进行赋值，如">"（大于）
- **condition.value**：阈值，通过引用输入变量进行赋值
- **condition.unit**：单位，通过引用输入变量进行赋值，如"%"（百分比）
- **condition.count**：连续触发次数，通过引用输入变量进行赋值
- **condition.alarm_level**：告警级别，通过引用输入变量进行赋值
- **condition.metric_name**：指标名称，通过引用输入变量进行赋值，如"cpu_util"（CPU使用率）
- **alarm_actions.type**：告警动作类型，设置为"autoscaling"（弹性伸缩）
- **alarm_actions.notification_list**：通知列表，设置为空数组

### 12. 创建伸缩策略资源

在TF文件（如main.tf）中添加以下脚本以创建伸缩策略：

```hcl
variable "policy_name" {
  description = "The name of the AS policy"
  type        = string
}

variable "policy_cool_down_time" {
  description = "The cool down time of the AS policy"
  type        = number
  default     = 900
}

variable "policy_operation" {
  description = "The operation of the AS policy"
  type        = string
  default     = "ADD"
}

variable "policy_instance_number" {
  description = "The instance number of the AS policy"
  type        = number
  default     = 1
}

# 创建伸缩策略
resource "huaweicloud_as_policy" "test" {
  scaling_policy_name = var.policy_name
  scaling_policy_type = "ALARM"
  scaling_group_id    = huaweicloud_as_group.test.id
  alarm_id            = huaweicloud_ces_alarmrule.test.id
  cool_down_time      = var.policy_cool_down_time

  scaling_policy_action {
    operation       = var.policy_operation
    instance_number = var.policy_instance_number
  }
}
```

**参数说明**：
- **scaling_policy_name**：伸缩策略名称，通过引用输入变量policy_name进行赋值
- **scaling_policy_type**：伸缩策略类型，设置为"ALARM"（告警策略）
- **scaling_group_id**：伸缩组ID，通过引用弹性伸缩组资源进行赋值
- **alarm_id**：告警规则ID，通过引用告警规则资源进行赋值
- **cool_down_time**：冷却时间，通过引用输入变量policy_cool_down_time进行赋值，默认值为900（秒）
- **scaling_policy_action.operation**：操作类型，通过引用输入变量policy_operation进行赋值，默认值为"ADD"（增加实例）
- **scaling_policy_action.instance_number**：实例数量，通过引用输入变量policy_instance_number进行赋值，默认值为1

### 13. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC和子网配置
vpc_name    = "tf_test_vpc"
vpc_cidr    = "172.16.0.0/16"
subnet_name = "tf_test_subnet"

# 安全组配置
security_group_name = "tf_test_security_group"

# 负载均衡器配置
loadbalancer_name              = "tf_test_dedicated_loadbalancer"
loadbalancer_cross_vpc_backend = true

# 监听器配置
listener_name = "tf_test_dedicated_listener"

# 后端服务器组配置
pool_name = "tf_test_dedicated_pool"

# 弹性伸缩配置
configuration_name = "tf_test_as_configuration"
configuration_user_data = <<EOT
#! /bin/bash
echo 'root:$6$V6azyeLwcD3CHlpY$BN3VVq18fmCkj66B4zdHLWevqcxlig' | chpasswd -e
EOT

configuration_disks = [
  {
    size        = 40
    volume_type = "SSD"
    disk_type   = "SYS"
  }
]

# 弹性伸缩组配置
group_name = "tf_test_as_group"

# ECS实例配置
instance_name = "tf_test_instance"

# 告警规则配置
alarm_rule_name = "tf_test_alarm_rule2"
alarm_rule_conditions = [
  {
    alarm_level         = 2
    period              = 300
    filter              = "max"
    comparison_operator = ">"
    value               = 80
    unit                = "%"
    count               = 1
    metric_name         = "cpu_util"
  }
]

# 伸缩策略配置
policy_name = "tf_test_as_policy"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `configuration_user_data`需要设置实例初始化脚本，用于配置实例的初始状态
   - `configuration_disks`需要配置磁盘信息，包括系统盘和数据盘
   - `alarm_rule_conditions`需要配置告警条件，如CPU使用率超过80%时触发告警
   - `policy_operation`可以设置为"ADD"（增加实例）或"REMOVE"（减少实例）
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="group_name=my_group" -var="loadbalancer_name=my_lb"`
2. 环境变量：`export TF_VAR_group_name=my_group` 和 `export TF_VAR_loadbalancer_name=my_lb`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。弹性伸缩组与负载均衡器关联后，弹性伸缩服务会自动将新创建的实例添加到负载均衡器的后端服务器组中。告警规则和伸缩策略配置后，当监控指标达到阈值时，弹性伸缩服务会自动执行伸缩操作。

### 14. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建专用负载均衡器与弹性伸缩的集成方案：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建VPC、子网、负载均衡器、弹性伸缩组、告警规则和伸缩策略等资源
4. 运行 `terraform show` 查看已创建的弹性伸缩组详情

> 注意：弹性伸缩组创建后，会根据配置自动创建或调整实例数量。当告警规则触发时，伸缩策略会自动执行，增加或减少实例数量。新创建的实例会自动添加到负载均衡器的后端服务器组中，实现自动化的流量分发和资源弹性伸缩。通过`lifecycle.ignore_changes`可以避免Terraform在后续更新时修改会自动变化的参数（如`desire_instance_number`）。

## 参考信息

- [华为云弹性负载均衡产品文档](https://support.huaweicloud.com/elb/index.html)
- [华为云弹性伸缩产品文档](https://support.huaweicloud.com/as/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [专用负载均衡器与弹性伸缩最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/elb/dedicated-loadbalancer-with-as)
