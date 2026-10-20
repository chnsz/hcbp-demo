# 部署DNAT规则

## 应用场景

NAT网关（NAT Gateway）是华为云提供的网络地址转换服务，支持SNAT和DNAT两种转换方式。其中DNAT（Destination Network Address Translation，目的地址转换）用于将公网EIP的指定端口映射到VPC内后端服务器的端口，使互联网上的客户端能够通过EIP访问VPC内的服务，而无需为后端服务器直接绑定公网IP。

本最佳实践将介绍如何使用Terraform自动化部署DNAT规则，包括VPC与子网创建、NAT网关创建、EIP创建、后端ECS实例与安全组配置，以及DNAT规则的创建与端口映射配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像列表（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [NAT网关（huaweicloud_nat_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_gateway)
- [弹性公网IP（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [DNAT规则（huaweicloud_nat_dnat_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_dnat_rule)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [弹性云服务器（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_compute_flavors
    │       └── data.huaweicloud_images_images
    │               └── huaweicloud_compute_instance
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            ├── huaweicloud_nat_gateway
            │       └── huaweicloud_nat_dnat_rule
            └── huaweicloud_compute_instance
                    └── huaweicloud_nat_dnat_rule

huaweicloud_vpc_eip
    └── huaweicloud_nat_dnat_rule

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_compute_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云和子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云和子网资源
variable "vpc_name" {
  description = "The VPC name"
  type        = string
  default     = "vpc-dnat-basic"
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "172.16.0.0/16"
}

variable "subnet_name" {
  description = "The subnet name"
  type        = string
  default     = "subnet-dnat-basic"
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "172.16.10.0/24"
}

variable "subnet_gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = true
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，通过引用输入变量 vpc_cidr 进行赋值
- **vpc_id**：子网所属的VPC ID，引用 `huaweicloud_vpc.test.id`
- **cidr**：子网网段，通过引用输入变量 subnet_cidr 进行赋值；未指定时由VPC网段自动计算
- **gateway_ip**：子网网关IP，通过引用输入变量 subnet_gateway_ip 进行赋值；未指定时由子网网段自动计算

### 3. 创建NAT网关

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建NAT网关资源
variable "nat_gateway_name" {
  description = "The NAT gateway name"
  type        = string
  default     = "nat-gateway-dnat-basic"
}

variable "nat_gateway_description" {
  description = "The description of the NAT gateway"
  type        = string
  default     = ""
}

variable "nat_gateway_spec" {
  description = "The specification of the NAT gateway"
  type        = string
  default     = "1"

  validation {
    condition     = contains(["1", "2", "3", "4"], var.nat_gateway_spec)
    error_message = "The nat_gateway_spec must be one of: 1, 2, 3, 4."
  }
}

resource "huaweicloud_nat_gateway" "test" {
  name        = var.nat_gateway_name
  description = var.nat_gateway_description
  spec        = var.nat_gateway_spec
  vpc_id      = huaweicloud_vpc.test.id
  subnet_id   = huaweicloud_vpc_subnet.test.id
}
```

**参数说明**：
- **name**：NAT网关名称，通过引用输入变量 nat_gateway_name 进行赋值
- **description**：NAT网关描述，通过引用输入变量 nat_gateway_description 进行赋值
- **spec**：NAT网关规格，通过引用输入变量 nat_gateway_spec 进行赋值，可选值为1（小型）、2（中型）、3（大型）、4（超大型）
- **vpc_id**：NAT网关所属的VPC ID，引用 `huaweicloud_vpc.test.id`
- **subnet_id**：NAT网关所属的子网ID，引用 `huaweicloud_vpc_subnet.test.id`

### 4. 创建弹性公网IP

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源
variable "eip_bandwidth_name" {
  description = "The name of the EIP bandwidth"
  type        = string
  default     = "eip-dnat-basic"
}

variable "eip_bandwidth_size" {
  description = "The size of the EIP bandwidth (Mbit/s)"
  type        = number
  default     = 5
}

variable "eip_bandwidth_share_type" {
  description = "The share type of the EIP bandwidth"
  type        = string
  default     = "PER"

  validation {
    condition     = contains(["PER", "WHOLE"], var.eip_bandwidth_share_type)
    error_message = "The eip_bandwidth_share_type must be one of: PER, WHOLE."
  }
}

variable "eip_bandwidth_charge_mode" {
  description = "The charge mode of the EIP bandwidth"
  type        = string
  default     = "traffic"

  validation {
    condition     = contains(["traffic", "bandwidth"], var.eip_bandwidth_charge_mode)
    error_message = "The eip_bandwidth_charge_mode must be one of: traffic, bandwidth."
  }
}

resource "huaweicloud_vpc_eip" "test" {
  publicip {
    type = "5_bgp"
  }

  bandwidth {
    name        = var.eip_bandwidth_name
    size        = var.eip_bandwidth_size
    share_type  = var.eip_bandwidth_share_type
    charge_mode = var.eip_bandwidth_charge_mode
  }
}
```

**参数说明**：
- **publicip.type**：EIP类型，固定为 `5_bgp`
- **bandwidth.name**：带宽名称，通过引用输入变量 eip_bandwidth_name 进行赋值
- **bandwidth.size**：带宽大小（Mbit/s），通过引用输入变量 eip_bandwidth_size 进行赋值
- **bandwidth.share_type**：带宽共享类型，通过引用输入变量 eip_bandwidth_share_type 进行赋值
- **bandwidth.charge_mode**：带宽计费模式，通过引用输入变量 eip_bandwidth_charge_mode 进行赋值

### 5. 查询可用区、ECS规格与镜像

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区、ECS规格与镜像数据源
data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_compute_flavors" "test" {
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.ecs_flavor_performance_type
  cpu_core_count    = var.ecs_flavor_cpu_core_count
  memory_size       = var.ecs_flavor_memory_size
}

data "huaweicloud_images_images" "test" {
  flavor_id  = var.ecs_flavor_id != "" ? var.ecs_flavor_id : try(data.huaweicloud_compute_flavors.test.flavors[0].id, null)
  visibility = var.ecs_image_visibility
  os         = var.ecs_image_os
}
```

**参数说明**：
- **availability_zone**：可用区名称，取可用区列表中的第一个可用区
- **performance_type**：ECS规格性能类型，通过引用输入变量 ecs_flavor_performance_type 进行赋值
- **cpu_core_count**：ECS规格CPU核数，通过引用输入变量 ecs_flavor_cpu_core_count 进行赋值
- **memory_size**：ECS规格内存大小（GB），通过引用输入变量 ecs_flavor_memory_size 进行赋值
- **flavor_id**：镜像查询所用的规格ID，优先使用输入变量 ecs_flavor_id，未指定时使用查询到的规格
- **visibility**：镜像可见性，通过引用输入变量 ecs_image_visibility 进行赋值
- **os**：镜像操作系统，通过引用输入变量 ecs_image_os 进行赋值

### 6. 创建安全组及安全组规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组及安全组规则资源
variable "security_group_name" {
  description = "The security group name of the backend instance"
  type        = string
  default     = "sg-dnat-backend"
}

variable "backend_protocol" {
  description = "The protocol used between the NAT gateway and backend ECS instance"
  type        = string
  default     = "tcp"
}

variable "backend_port" {
  description = "The port on the backend ECS instance that receives DNAT traffic"
  type        = number
  default     = 22
}

variable "ingress_cidr" {
  description = "The CIDR block that is allowed to access the DNAT service from the Internet"
  type        = string
  default     = "0.0.0.0/0"
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "ingress" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = var.backend_protocol
  port_range_min    = var.backend_port
  port_range_max    = var.backend_port
  remote_ip_prefix  = var.ingress_cidr
}

resource "huaweicloud_networking_secgroup_rule" "egress" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "egress"
  ethertype         = "IPv4"
  remote_ip_prefix  = "0.0.0.0/0"
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：是否删除默认规则，设置为 `true` 以便仅保留自定义规则
- **direction**：规则方向，入方向为 `ingress`，出方向为 `egress`
- **protocol**：入方向规则协议，通过引用输入变量 backend_protocol 进行赋值
- **port_range_min/port_range_max**：入方向规则端口范围，通过引用输入变量 backend_port 进行赋值
- **remote_ip_prefix**：入方向规则源地址网段，通过引用输入变量 ingress_cidr 进行赋值

### 7. 创建后端ECS实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建后端ECS实例资源
variable "instance_name" {
  description = "The name of the backend ECS instance"
  type        = string
  default     = "ecs-dnat-backend"
}

variable "ecs_flavor_id" {
  description = "The flavor ID of the backend ECS instance"
  type        = string
  default     = ""
  nullable    = true
}

variable "ecs_image_id" {
  description = "The image ID of the backend ECS instance"
  type        = string
  default     = ""
  nullable    = true
}

variable "ecs_system_disk_type" {
  description = "The system disk type of the ECS instance"
  type        = string
  default     = "SSD"
}

variable "ecs_system_disk_size" {
  description = "The system disk size of the ECS instance (GB)"
  type        = number
  default     = 40
}

variable "ecs_admin_password" {
  description = "The administrator password of the ECS instance"
  type        = string
  sensitive   = true
  default     = ""
}

variable "ecs_instance_tags" {
  description = "The tags of the ECS instance"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_compute_instance" "test" {
  name               = var.instance_name
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)
  flavor_id          = var.ecs_flavor_id != "" ? var.ecs_flavor_id : try(data.huaweicloud_compute_flavors.test.flavors[0].id, "")
  image_id           = var.ecs_image_id != "" ? var.ecs_image_id : try(data.huaweicloud_images_images.test.images[0].id, "")
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  system_disk_type   = var.ecs_system_disk_type
  system_disk_size   = var.ecs_system_disk_size

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  admin_pass = var.ecs_admin_password
  tags       = var.ecs_instance_tags

  depends_on = [
    huaweicloud_nat_gateway.test,
    huaweicloud_vpc_eip.test
  ]
}
```

**参数说明**：
- **name**：ECS实例名称，通过引用输入变量 instance_name 进行赋值
- **availability_zone**：ECS实例所在可用区，取可用区列表中的第一个可用区
- **flavor_id**：ECS实例规格ID，优先使用输入变量 ecs_flavor_id，未指定时使用查询到的规格
- **image_id**：ECS实例镜像ID，优先使用输入变量 ecs_image_id，未指定时使用查询到的镜像
- **security_group_ids**：ECS实例绑定的安全组ID列表，引用 `huaweicloud_networking_secgroup.test.id`
- **system_disk_type**：系统盘类型，通过引用输入变量 ecs_system_disk_type 进行赋值
- **system_disk_size**：系统盘大小（GB），通过引用输入变量 ecs_system_disk_size 进行赋值
- **network.uuid**：ECS实例所属子网ID，引用 `huaweicloud_vpc_subnet.test.id`
- **admin_pass**：ECS实例管理员密码，通过引用输入变量 ecs_admin_password 进行赋值
- **tags**：ECS实例标签，通过引用输入变量 ecs_instance_tags 进行赋值

### 8. 创建DNAT规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DNAT规则资源
variable "frontend_protocol" {
  description = "The protocol used between the client and NAT gateway"
  type        = string
  default     = "tcp"

  validation {
    condition     = contains(["tcp", "udp", "any"], var.frontend_protocol)
    error_message = "The frontend_protocol must be one of: tcp, udp, any."
  }
}

variable "frontend_port" {
  description = "The port on the public EIP that clients use to access the DNAT service"
  type        = number
  default     = 22
}

resource "huaweicloud_nat_dnat_rule" "test" {
  nat_gateway_id        = huaweicloud_nat_gateway.test.id
  floating_ip_id        = huaweicloud_vpc_eip.test.id
  port_id               = try(huaweicloud_compute_instance.test.network[0].port, null)
  protocol              = var.frontend_protocol
  internal_service_port = var.backend_port
  external_service_port = var.frontend_port
}
```

**参数说明**：
- **nat_gateway_id**：DNAT规则所属的NAT网关ID，引用 `huaweicloud_nat_gateway.test.id`
- **floating_ip_id**：DNAT规则绑定的EIP ID，引用 `huaweicloud_vpc_eip.test.id`
- **port_id**：后端ECS实例的网卡端口ID，引用 `huaweicloud_compute_instance.test.network[0].port`
- **protocol**：客户端与NAT网关之间的协议，通过引用输入变量 frontend_protocol 进行赋值
- **internal_service_port**：后端ECS实例接收DNAT流量的端口，通过引用输入变量 backend_port 进行赋值
- **external_service_port**：客户端访问DNAT服务所使用的EIP端口，通过引用输入变量 frontend_port 进行赋值

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC和子网
vpc_name          = "vpc-dnat-basic"
vpc_cidr          = "172.16.0.0/16"
subnet_name       = "subnet-dnat-basic"
subnet_cidr       = "172.16.0.0/24"
subnet_gateway_ip = "172.16.0.1"

# 后端ECS
security_group_name = "sg-dnat-backend"
ingress_cidr        = "192.168.0.0/16"
instance_name       = "ecs-dnat-backend"

# 可选：配置数据源查询参数（未指定ID时使用）
ecs_flavor_performance_type = "normal"
ecs_flavor_cpu_core_count   = 2
ecs_flavor_memory_size      = 4
ecs_image_visibility        = "public"
ecs_image_os                = "Ubuntu"
ecs_system_disk_type        = "SSD"
ecs_system_disk_size        = 40

# DNAT端口与协议
backend_protocol  = "tcp"
backend_port      = 2288
frontend_protocol = "tcp"
frontend_port     = 2288

# EIP和NAT网关
eip_bandwidth_name        = "eip-dnat-basic"
eip_bandwidth_size        = 5
eip_bandwidth_share_type  = "PER"
eip_bandwidth_charge_mode = "traffic"
nat_gateway_name          = "nat-gateway-dnat-basic"
nat_gateway_description   = ""
nat_gateway_spec          = "1"
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

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DNAT规则
4. 运行 `terraform show` 查看已创建的DNAT规则

## 参考信息

- [华为云NAT网关产品文档](https://support.huaweicloud.com/natgateway/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [NAT网关DNAT规则最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/nat/dnat-basic)
