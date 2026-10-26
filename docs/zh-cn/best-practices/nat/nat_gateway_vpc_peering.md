# 部署通过VPC对等连接共享网关

## 应用场景

NAT网关（NAT Gateway）是华为云提供的网络地址转换服务，能够为虚拟私有云（VPC）内的云资源提供访问公网以及被公网访问的能力。当企业存在多个VPC且希望共享同一套公网出口与入方向映射时，可以在一个VPC中部署NAT网关，再通过VPC对等连接将其他VPC的流量引导至该NAT网关，从而减少EIP与NAT网关资源的重复建设，降低公网访问成本。

本最佳实践将介绍如何使用Terraform在VPC One中创建NAT网关、EIP、SNAT规则与DNAT规则，并通过VPC对等连接与路由配置，使VPC Other中的ECS实例能够借助VPC One的NAT网关访问互联网，同时支持通过DNAT规则从公网访问两个VPC中的ECS实例。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [镜像（data.huaweicloud_images_image）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_image)
- [计算规格（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### 资源

- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [弹性云服务器（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [VPC对等连接（huaweicloud_vpc_peering_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_peering_connection)
- [VPC路由（huaweicloud_vpc_route）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_route)
- [弹性公网IP（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [NAT网关（huaweicloud_nat_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_gateway)
- [SNAT规则（huaweicloud_nat_snat_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_snat_rule)
- [DNAT规则（huaweicloud_nat_dnat_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_dnat_rule)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_compute_instance

data.huaweicloud_images_image
    └── huaweicloud_compute_instance

data.huaweicloud_compute_flavors
    └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
    └── huaweicloud_vpc_peering_connection
    └── huaweicloud_vpc_route
    └── huaweicloud_nat_gateway

huaweicloud_vpc_subnet
    └── huaweicloud_compute_instance
    └── huaweicloud_nat_gateway
    └── huaweicloud_nat_snat_rule

huaweicloud_vpc_peering_connection
    └── huaweicloud_vpc_route

huaweicloud_vpc_eip
    └── huaweicloud_nat_snat_rule
    └── huaweicloud_nat_dnat_rule

huaweicloud_nat_gateway
    └── huaweicloud_nat_snat_rule
    └── huaweicloud_nat_dnat_rule

huaweicloud_compute_instance
    └── huaweicloud_nat_dnat_rule
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区、镜像与计算规格

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区、镜像与计算规格信息
variable "image_name" {
  description = "The name of the image to use for ECS instances"
  type        = string
  default     = "Ubuntu 22.04 server 64bit"
}

variable "image_visibility" {
  description = "The visibility of the image"
  type        = string
  default     = "public"
}

variable "image_most_recent" {
  description = "If more than one result is returned, use the most recent image"
  type        = bool
  default     = true
}

variable "flavor_performance_type" {
  description = "The performance type of the flavor"
  type        = string
  default     = "normal"
}

variable "flavor_cpu_core_count" {
  description = "The number of CPU cores for the flavor"
  type        = number
  default     = 2
}

variable "flavor_memory_size" {
  description = "The memory size in GB for the flavor"
  type        = number
  default     = 4
}

data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_images_image" "test" {
  name        = var.image_name
  visibility  = var.image_visibility
  most_recent = var.image_most_recent
}

data "huaweicloud_compute_flavors" "test" {
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.flavor_performance_type
  cpu_core_count    = var.flavor_cpu_core_count
  memory_size       = var.flavor_memory_size
}
```

**参数说明**：
- **image_name**：通过引用输入变量 image_name 进行赋值，用于指定ECS实例使用的镜像名称
- **image_visibility**：通过引用输入变量 image_visibility 进行赋值，用于指定镜像的可见性
- **image_most_recent**：通过引用输入变量 image_most_recent 进行赋值，用于指定当返回多个镜像时是否使用最新的镜像
- **flavor_performance_type**：通过引用输入变量 flavor_performance_type 进行赋值，用于指定计算规格的性能类型
- **flavor_cpu_core_count**：通过引用输入变量 flavor_cpu_core_count 进行赋值，用于指定计算规格的CPU核数
- **flavor_memory_size**：通过引用输入变量 flavor_memory_size 进行赋值，用于指定计算规格的内存大小

### 3. 创建安全组及安全组规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组及安全组规则
variable "secgroup_name" {
  description = "The name of the security group"
  type        = string
  default     = "secgroup-example"
}

variable "security_group_rules" {
  description = "List of security group rules"
  type        = list(object({
    direction        = string
    description      = string
    ethertype        = string
    protocol         = string
    ports            = optional(string)
    remote_ip_prefix = string
  }))
  default     = [
    {
      direction        = "ingress"
      description      = "Allow SSH access"
      ethertype        = "IPv4"
      protocol         = "tcp"
      ports            = "22"
      remote_ip_prefix = "0.0.0.0/0"
    },
    {
      direction        = "ingress"
      description      = "Allow HTTP and HTTPS access"
      ethertype        = "IPv4"
      protocol         = "tcp"
      ports            = "80,443"
      remote_ip_prefix = "0.0.0.0/0"
    },
    {
      direction        = "ingress"
      description      = "Allow ICMP access"
      ethertype        = "IPv4"
      protocol         = "icmp"
      ports            = null
      remote_ip_prefix = "0.0.0.0/0"
    }
  ]
}

resource "huaweicloud_networking_secgroup" "test" {
  name        = var.secgroup_name
  description = "Security group for ECS instances"
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = length(var.security_group_rules)

  direction         = var.security_group_rules[count.index].direction
  description       = var.security_group_rules[count.index].description
  ethertype         = var.security_group_rules[count.index].ethertype
  protocol          = var.security_group_rules[count.index].protocol
  ports             = var.security_group_rules[count.index].ports
  remote_ip_prefix  = var.security_group_rules[count.index].remote_ip_prefix
  security_group_id = huaweicloud_networking_secgroup.test.id
}
```

**参数说明**：
- **name**：通过引用输入变量 secgroup_name 进行赋值，用于指定安全组名称
- **direction**：通过引用输入变量 security_group_rules 中的 direction 进行赋值，用于指定规则方向
- **description**：通过引用输入变量 security_group_rules 中的 description 进行赋值，用于指定规则描述
- **ethertype**：通过引用输入变量 security_group_rules 中的 ethertype 进行赋值，用于指定IP协议类型
- **protocol**：通过引用输入变量 security_group_rules 中的 protocol 进行赋值，用于指定协议类型
- **ports**：通过引用输入变量 security_group_rules 中的 ports 进行赋值，用于指定端口范围
- **remote_ip_prefix**：通过引用输入变量 security_group_rules 中的 remote_ip_prefix 进行赋值，用于指定远端IP地址范围
- **security_group_id**：通过引用安全组资源 huaweicloud_networking_secgroup.test 的ID进行赋值

### 4. 创建VPC与子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC与子网
variable "vpcs" {
  description = "List of VPC configurations"
  type        = list(object({
    vpc_name             = string
    vpc_cidr             = string
    subnet_name          = string
    subnet_cidr          = string
    subnet_gateway_ip    = optional(string)
    instance_name        = string
    ecs_flavor_id        = optional(string)
    ecs_image_id         = optional(string)
    ecs_system_disk_type = optional(string)
    ecs_system_disk_size = optional(number)
    ecs_admin_password   = optional(string)
    ecs_instance_tags    = optional(map(string))
  }))
  default     = [
    {
      vpc_name             = "vpc-one"
      vpc_cidr             = "192.168.0.0/16"
      subnet_name          = "subnet-one"
      subnet_cidr          = "192.168.1.0/24"
      subnet_gateway_ip    = "192.168.1.1"
      instance_name        = "instance-one"
      ecs_flavor_id        = null
      ecs_image_id         = null
      ecs_system_disk_type = null
      ecs_system_disk_size = null
      ecs_admin_password   = null
      ecs_instance_tags    = null
    },
    {
      vpc_name             = "vpc-other"
      vpc_cidr             = "10.0.0.0/16"
      subnet_name          = "subnet-other"
      subnet_cidr          = "10.0.2.0/24"
      subnet_gateway_ip    = "10.0.2.1"
      instance_name        = "instance-other"
      ecs_flavor_id        = null
      ecs_image_id         = null
      ecs_system_disk_type = null
      ecs_system_disk_size = null
      ecs_admin_password   = null
      ecs_instance_tags    = null
    }
  ]
}

resource "huaweicloud_vpc" "test" {
  count = length(var.vpcs)

  name = var.vpcs[count.index].vpc_name
  cidr = var.vpcs[count.index].vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  count = length(var.vpcs)

  vpc_id     = huaweicloud_vpc.test[count.index].id
  name       = var.vpcs[count.index].subnet_name
  cidr       = var.vpcs[count.index].subnet_cidr
  gateway_ip = var.vpcs[count.index].subnet_gateway_ip != null && var.vpcs[count.index].subnet_gateway_ip != "" ? var.vpcs[count.index].subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test[count.index].cidr, 8, 0), 1)
}
```

**参数说明**：
- **name**：通过引用输入变量 vpcs 中的 vpc_name 进行赋值，用于指定VPC名称
- **cidr**：通过引用输入变量 vpcs 中的 vpc_cidr 进行赋值，用于指定VPC网段
- **vpc_id**：通过引用VPC资源 huaweicloud_vpc.test 的ID进行赋值
- **name**（子网）：通过引用输入变量 vpcs 中的 subnet_name 进行赋值，用于指定子网名称
- **cidr**（子网）：通过引用输入变量 vpcs 中的 subnet_cidr 进行赋值，用于指定子网网段
- **gateway_ip**：通过引用输入变量 vpcs 中的 subnet_gateway_ip 进行赋值，未指定时自动计算子网网关IP

### 5. 创建ECS实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例
variable "ecs_admin_password" {
  description = "The administrator password of ECS instances (used as fallback when not specified in vpcs)"
  type        = string
  sensitive   = true
}

resource "huaweicloud_compute_instance" "test" {
  count = length(var.vpcs)

  name               = var.vpcs[count.index].instance_name
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)
  flavor_id          = var.vpcs[count.index].ecs_flavor_id != null && var.vpcs[count.index].ecs_flavor_id != "" ? var.vpcs[count.index].ecs_flavor_id : try(data.huaweicloud_compute_flavors.test.ids[0], null)
  image_id           = var.vpcs[count.index].ecs_image_id != null && var.vpcs[count.index].ecs_image_id != "" ? var.vpcs[count.index].ecs_image_id : try(data.huaweicloud_images_image.test.id, null)
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  system_disk_type   = var.vpcs[count.index].ecs_system_disk_type != null ? var.vpcs[count.index].ecs_system_disk_type : "SSD"
  system_disk_size   = var.vpcs[count.index].ecs_system_disk_size != null ? var.vpcs[count.index].ecs_system_disk_size : 40

  network {
    uuid = huaweicloud_vpc_subnet.test[count.index].id
  }

  admin_pass = var.vpcs[count.index].ecs_admin_password != null && var.vpcs[count.index].ecs_admin_password != "" ? var.vpcs[count.index].ecs_admin_password : var.ecs_admin_password
  tags       = var.vpcs[count.index].ecs_instance_tags != null ? var.vpcs[count.index].ecs_instance_tags : {}

  depends_on = [
    huaweicloud_nat_gateway.test,
    huaweicloud_vpc_eip.test
  ]
}
```

**参数说明**：
- **name**：通过引用输入变量 vpcs 中的 instance_name 进行赋值，用于指定ECS实例名称
- **availability_zone**：通过引用可用区数据源 data.huaweicloud_availability_zones.test 的第一个可用区进行赋值
- **flavor_id**：优先使用输入变量 vpcs 中的 ecs_flavor_id，未指定时使用计算规格数据源返回的第一个规格ID
- **image_id**：优先使用输入变量 vpcs 中的 ecs_image_id，未指定时使用镜像数据源返回的镜像ID
- **security_group_ids**：通过引用安全组资源 huaweicloud_networking_secgroup.test 的ID进行赋值
- **system_disk_type**：通过引用输入变量 vpcs 中的 ecs_system_disk_type 进行赋值，未指定时默认为SSD
- **system_disk_size**：通过引用输入变量 vpcs 中的 ecs_system_disk_size 进行赋值，未指定时默认为40GB
- **network.uuid**：通过引用子网资源 huaweicloud_vpc_subnet.test 的ID进行赋值
- **admin_pass**：优先使用输入变量 vpcs 中的 ecs_admin_password，未指定时使用全局输入变量 ecs_admin_password
- **tags**：通过引用输入变量 vpcs 中的 ecs_instance_tags 进行赋值

### 6. 创建VPC对等连接与路由

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC对等连接与路由
variable "peering_connection_name" {
  description = "The name of the VPC peering connection"
  type        = string
  default     = "peering-connection-example"
}

variable "route_type" {
  description = "The type of the route"
  type        = string
  default     = "peering"
}

resource "huaweicloud_vpc_peering_connection" "cross_vpc_peering" {
  name        = var.peering_connection_name
  vpc_id      = huaweicloud_vpc.test[0].id
  peer_vpc_id = huaweicloud_vpc.test[1].id
}

resource "huaweicloud_vpc_route" "route_to_peer_vpc" {
  vpc_id      = huaweicloud_vpc.test[0].id
  destination = try(var.vpcs[1].vpc_cidr, null)
  type        = var.route_type
  nexthop     = huaweicloud_vpc_peering_connection.cross_vpc_peering.id
}

resource "huaweicloud_vpc_route" "route_other_to_internet_via_peering" {
  vpc_id      = huaweicloud_vpc.test[1].id
  destination = "0.0.0.0/0"
  type        = var.route_type
  nexthop     = huaweicloud_vpc_peering_connection.cross_vpc_peering.id
}
```

**参数说明**：
- **name**：通过引用输入变量 peering_connection_name 进行赋值，用于指定对等连接名称
- **vpc_id**：通过引用VPC资源 huaweicloud_vpc.test[0] 的ID进行赋值，表示本端VPC
- **peer_vpc_id**：通过引用VPC资源 huaweicloud_vpc.test[1] 的ID进行赋值，表示对端VPC
- **destination**：在VPC One中通过引用输入变量 vpcs[1].vpc_cidr 进行赋值，在VPC Other中固定为 0.0.0.0/0
- **type**：通过引用输入变量 route_type 进行赋值，用于指定路由类型
- **nexthop**：通过引用对等连接资源 huaweicloud_vpc_peering_connection.cross_vpc_peering 的ID进行赋值

### 7. 创建EIP

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建EIP
variable "eips" {
  description = "List of EIP configurations"
  type        = list(object({
    name_suffix = string
  }))
  default     = [
    {
      name_suffix = "snat"
    },
    {
      name_suffix = "dnat"
    }
  ]
}

variable "nat_gateway_name" {
  description = "The name of the NAT gateway"
  type        = string
  default     = "nat-gateway-example"
}

variable "bandwidth_size" {
  description = "The bandwidth size in Mbps"
  type        = number
  default     = 5
}

variable "bandwidth_charge_mode" {
  description = "The charge mode of the bandwidth"
  type        = string
  default     = "traffic"
}

resource "huaweicloud_vpc_eip" "test" {
  count = length(var.eips)

  publicip {
    type = "5_bgp"
  }

  bandwidth {
    share_type  = "PER"
    name        = "${var.nat_gateway_name}-${var.eips[count.index].name_suffix}-shared-bandwidth"
    size        = var.bandwidth_size
    charge_mode = var.bandwidth_charge_mode
  }
}
```

**参数说明**：
- **publicip.type**：固定为 5_bgp，表示使用全动态BGP类型的公网IP
- **bandwidth.share_type**：固定为 PER，表示使用独享带宽
- **bandwidth.name**：由输入变量 nat_gateway_name 与 eips 中的 name_suffix 组合赋值，用于标识带宽名称
- **bandwidth.size**：通过引用输入变量 bandwidth_size 进行赋值，用于指定带宽大小
- **bandwidth.charge_mode**：通过引用输入变量 bandwidth_charge_mode 进行赋值，用于指定带宽计费模式

### 8. 创建NAT网关

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建NAT网关
variable "nat_gateway_spec" {
  description = "The specification of the NAT gateway"
  type        = string
  default     = "1"
}

resource "huaweicloud_nat_gateway" "test" {
  name        = var.nat_gateway_name
  description = "NAT gateway for VPC One to provide internet access"
  spec        = var.nat_gateway_spec
  vpc_id      = huaweicloud_vpc.test[0].id
  subnet_id   = huaweicloud_vpc_subnet.test[0].id
}
```

**参数说明**：
- **name**：通过引用输入变量 nat_gateway_name 进行赋值，用于指定NAT网关名称
- **description**：用于描述NAT网关用途
- **spec**：通过引用输入变量 nat_gateway_spec 进行赋值，用于指定NAT网关规格
- **vpc_id**：通过引用VPC资源 huaweicloud_vpc.test[0] 的ID进行赋值，表示NAT网关部署在VPC One中
- **subnet_id**：通过引用子网资源 huaweicloud_vpc_subnet.test[0] 的ID进行赋值

### 9. 创建SNAT规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SNAT规则
variable "snat_rules" {
  description = "List of SNAT rule configurations"
  type        = list(object({
    eip_index    = number
    subnet_index = optional(number)
    source_type  = optional(number)
    vpc_index    = optional(number)
  }))
  default     = [
    {
      eip_index    = 0
      subnet_index = 0
      source_type  = null
      vpc_index    = null
    },
    {
      eip_index    = 0
      subnet_index = null
      source_type  = 1
      vpc_index    = 1
    }
  ]
}

resource "huaweicloud_nat_snat_rule" "test" {
  count = length(var.snat_rules)

  nat_gateway_id = huaweicloud_nat_gateway.test.id
  floating_ip_id = huaweicloud_vpc_eip.test[var.snat_rules[count.index].eip_index].id
  subnet_id      = var.snat_rules[count.index].subnet_index != null ? huaweicloud_vpc_subnet.test[var.snat_rules[count.index].subnet_index].id : null
  source_type    = var.snat_rules[count.index].source_type
  cidr           = var.snat_rules[count.index].vpc_index != null ? var.vpcs[var.snat_rules[count.index].vpc_index].subnet_cidr : null
}
```

**参数说明**：
- **nat_gateway_id**：通过引用NAT网关资源 huaweicloud_nat_gateway.test 的ID进行赋值
- **floating_ip_id**：通过引用EIP资源 huaweicloud_vpc_eip.test 中由 eip_index 指定的EIP ID进行赋值
- **subnet_id**：当 snat_rules 中指定 subnet_index 时，通过引用对应子网资源 huaweicloud_vpc_subnet.test 的ID进行赋值
- **source_type**：通过引用输入变量 snat_rules 中的 source_type 进行赋值，用于指定SNAT规则源类型
- **cidr**：当 snat_rules 中指定 vpc_index 时，通过引用输入变量 vpcs 中对应子网网段进行赋值

### 10. 创建DNAT规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DNAT规则
variable "dnat_rules" {
  description = "List of DNAT rule configurations"
  type        = list(object({
    eip_index             = number
    instance_index        = number
    external_service_port = string
  }))
  default     = [
    {
      eip_index             = 1
      instance_index        = 0
      external_service_port = "8022"
    },
    {
      eip_index             = 1
      instance_index        = 1
      external_service_port = "8023"
    }
  ]
}

variable "dnat_protocol" {
  description = "The protocol for DNAT rules"
  type        = string
  default     = "tcp"
}

variable "dnat_internal_port" {
  description = "The internal service port for DNAT rules"
  type        = string
  default     = "22"
}

resource "huaweicloud_nat_dnat_rule" "test" {
  count = length(var.dnat_rules)

  nat_gateway_id        = huaweicloud_nat_gateway.test.id
  floating_ip_id        = huaweicloud_vpc_eip.test[var.dnat_rules[count.index].eip_index].id
  private_ip            = try(huaweicloud_compute_instance.test[var.dnat_rules[count.index].instance_index].network[0].fixed_ip_v4, null)
  protocol              = var.dnat_protocol
  internal_service_port = var.dnat_internal_port
  external_service_port = var.dnat_rules[count.index].external_service_port
}
```

**参数说明**：
- **nat_gateway_id**：通过引用NAT网关资源 huaweicloud_nat_gateway.test 的ID进行赋值
- **floating_ip_id**：通过引用EIP资源 huaweicloud_vpc_eip.test 中由 eip_index 指定的EIP ID进行赋值
- **private_ip**：通过引用ECS实例资源 huaweicloud_compute_instance.test 中由 instance_index 指定的实例私网IP进行赋值
- **protocol**：通过引用输入变量 dnat_protocol 进行赋值，用于指定DNAT规则协议
- **internal_service_port**：通过引用输入变量 dnat_internal_port 进行赋值，用于指定内部服务端口
- **external_service_port**：通过引用输入变量 dnat_rules 中的 external_service_port 进行赋值，用于指定外部服务端口

### 11. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证与区域信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# ECS实例管理员密码
ecs_admin_password = "YourSecurePassword123!"

# VPC配置
vpcs = [
  {
    vpc_name          = "my-vpc-one"
    vpc_cidr          = "192.168.0.0/16"
    subnet_name       = "my-subnet-one"
    subnet_cidr       = "192.168.1.0/24"
    subnet_gateway_ip = "192.168.1.1"
    instance_name     = "my-instance-one"
  },
  {
    vpc_name          = "my-vpc-other"
    vpc_cidr          = "10.0.0.0/16"
    subnet_name       = "my-subnet-other"
    subnet_cidr       = "10.0.2.0/24"
    subnet_gateway_ip = "10.0.2.1"
    instance_name     = "my-instance-other"
  }
]

# NAT网关配置
nat_gateway_name = "my-nat-gateway"
bandwidth_size   = 10
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="nat_gateway_name=my-nat-gateway"`
2. 环境变量：`export TF_VAR_nat_gateway_name=my-nat-gateway`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 12. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建NAT网关及VPC对等连接
4. 运行 `terraform show` 查看已创建的NAT网关及VPC对等连接

## 参考信息

- [华为云NAT网关产品文档](https://support.huaweicloud.com/natgateway/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [NAT网关与VPC对等连接最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/nat/nat-gateway-vpc-peering)
