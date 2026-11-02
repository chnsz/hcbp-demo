# 部署SNAT规则

## 应用场景

NAT网关（NAT Gateway）是华为云提供的网络地址转换服务，能够为虚拟私有云（VPC）内的云资源提供访问公网的能力，而无需为每台服务器单独绑定公网IP。SNAT（Source Network Address Translation）规则通过将VPC内子网或指定网段的流量转换为NAT网关绑定的弹性公网IP，使内部实例可以共享公网出口访问互联网，同时隐藏内部网络地址。

本最佳实践将介绍如何使用Terraform自动化部署SNAT规则，包括VPC与子网创建、NAT网关创建、EIP创建、SNAT规则配置以及用于验证出方向访问的ECS实例与安全组配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [计算规格列表（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像列表（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [NAT网关（huaweicloud_nat_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_gateway)
- [弹性公网IP（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [SNAT规则（huaweicloud_nat_snat_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_snat_rule)
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
            │       └── huaweicloud_nat_snat_rule
            └── huaweicloud_compute_instance

huaweicloud_vpc_eip
    └── huaweicloud_nat_snat_rule

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
    └── huaweicloud_compute_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC和子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC和子网资源
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC网段，通过引用输入变量 vpc_cidr 进行赋值
- **vpc_id**：子网所属的VPC ID，引用 `huaweicloud_vpc.test.id`
- **name**（子网）：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**（子网）：子网网段，通过引用输入变量 subnet_cidr 进行赋值
- **gateway_ip**：子网网关IP，当输入变量 subnet_gateway_ip 为空时，基于VPC网段自动计算子网网关地址

### 3. 创建NAT网关

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建NAT网关资源
variable "nat_gateway_name" {
  description = "The name of the NAT gateway"
  type        = string
}

variable "gateway_description" {
  description = "The description of the NAT gateway"
  type        = string
  default     = ""
}

variable "gateway_spec" {
  description = "The specification of the NAT gateway"
  type        = string
  default     = "1"

  validation {
    condition     = contains(["1", "2", "3", "4"], var.gateway_spec)
    error_message = "The gateway_spec must be one of: 1, 2, 3, 4."
  }
}

resource "huaweicloud_nat_gateway" "test" {
  name        = var.nat_gateway_name
  description = var.gateway_description
  spec        = var.gateway_spec
  vpc_id      = huaweicloud_vpc.test.id
  subnet_id   = huaweicloud_vpc_subnet.test.id
}
```

**参数说明**：
- **name**：NAT网关名称，通过引用输入变量 nat_gateway_name 进行赋值
- **description**：NAT网关描述，通过引用输入变量 gateway_description 进行赋值
- **spec**：NAT网关规格，通过引用输入变量 gateway_spec 进行赋值，可选值为1（小型）、2（中型）、3（大型）、4（超大型）
- **vpc_id**：NAT网关所属的VPC ID，引用 `huaweicloud_vpc.test.id`
- **subnet_id**：NAT网关所属的子网ID，引用 `huaweicloud_vpc_subnet.test.id`

### 4. 创建弹性公网IP

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源
variable "eip_bandwidth_name" {
  description = "The name of the EIP bandwidth"
  type        = string
}

variable "eip_bandwidth_size" {
  description = "The size of the EIP bandwidth in Mbps"
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
- **type**：弹性公网IP类型，固定为 `5_bgp`
- **name**：带宽名称，通过引用输入变量 eip_bandwidth_name 进行赋值
- **size**：带宽大小（Mbps），通过引用输入变量 eip_bandwidth_size 进行赋值
- **share_type**：带宽共享类型，通过引用输入变量 eip_bandwidth_share_type 进行赋值，可选值为 PER（独享）、WHOLE（共享）
- **charge_mode**：带宽计费模式，通过引用输入变量 eip_bandwidth_charge_mode 进行赋值，可选值为 traffic（按流量）、bandwidth（按带宽）

### 5. 创建SNAT规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SNAT规则资源
variable "snat_source_type" {
  description = "The resource type of the SNAT rule"
  type        = number
  default     = 0
  nullable    = false

  validation {
    condition     = contains([0, 1], var.snat_source_type)
    error_message = "The snat_source_type must be one of: 0 (VPC scenario), 1 (Direct Connect scenario)."
  }
}

variable "snat_cidr" {
  description = "The CIDR block connected by SNAT rule (DC side, required when snat_source_type is 1)"
  type        = string
  default     = ""
  nullable    = false
}

variable "snat_description" {
  description = "The description of the SNAT rule"
  type        = string
  default     = ""
}

resource "huaweicloud_nat_snat_rule" "test" {
  nat_gateway_id = huaweicloud_nat_gateway.test.id
  floating_ip_id = huaweicloud_vpc_eip.test.id
  source_type    = var.snat_source_type
  subnet_id      = var.snat_source_type == 0 ? huaweicloud_vpc_subnet.test.id : null
  cidr           = var.snat_source_type == 1 ? var.snat_cidr : null
  description    = var.snat_description
}
```

**参数说明**：
- **nat_gateway_id**：SNAT规则所属的NAT网关ID，引用 `huaweicloud_nat_gateway.test.id`
- **floating_ip_id**：SNAT规则使用的弹性公网IP ID，引用 `huaweicloud_vpc_eip.test.id`
- **source_type**：SNAT规则源类型，通过引用输入变量 snat_source_type 进行赋值，0表示VPC场景，1表示云专线场景
- **subnet_id**：当 source_type 为0时，通过引用 `huaweicloud_vpc_subnet.test.id` 指定子网
- **cidr**：当 source_type 为1时，通过引用输入变量 snat_cidr 指定云专线侧网段
- **description**：SNAT规则描述，通过引用输入变量 snat_description 进行赋值

### 6. 查询可用区、计算规格和镜像

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区、计算规格和镜像数据源
variable "ecs_flavor_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "ecs_flavor_cpu_core_count" {
  description = "The CPU core count of the ECS instance flavor"
  type        = number
  default     = 2
}

variable "ecs_flavor_memory_size" {
  description = "The memory size of the ECS instance flavor"
  type        = number
  default     = 4
}

variable "ecs_flavor_id" {
  description = "The ID of the ECS instance flavor"
  type        = string
  default     = ""
  nullable    = false
}

variable "ecs_image_visibility" {
  description = "The visibility of the ECS instance image"
  type        = string
  default     = "public"
}

variable "ecs_image_os" {
  description = "The OS of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

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
- **availability_zone**：计算规格查询使用的可用区，取可用区列表数据源的第一个可用区
- **performance_type**：计算规格性能类型，通过引用输入变量 ecs_flavor_performance_type 进行赋值
- **cpu_core_count**：计算规格CPU核数，通过引用输入变量 ecs_flavor_cpu_core_count 进行赋值
- **memory_size**：计算规格内存大小（GB），通过引用输入变量 ecs_flavor_memory_size 进行赋值
- **flavor_id**：镜像查询使用的计算规格ID，优先使用输入变量 ecs_flavor_id，为空时取计算规格列表数据源的第一个规格
- **visibility**：镜像可见性，通过引用输入变量 ecs_image_visibility 进行赋值
- **os**：镜像操作系统，通过引用输入变量 ecs_image_os 进行赋值

### 7. 创建安全组和规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组和规则资源
variable "ecs_security_group_name" {
  description = "The name of the security group for ECS instance"
  type        = string
  default     = "terraform-test-ecs-sg"
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.ecs_security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "egress"
  ethertype         = "IPv4"
  remote_ip_prefix  = "0.0.0.0/0"
  priority          = 1
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量 ecs_security_group_name 进行赋值
- **delete_default_rules**：是否删除安全组默认规则，设置为 `true`
- **security_group_id**：安全组规则所属的安全组ID，引用 `huaweicloud_networking_secgroup.test.id`
- **direction**：规则方向，设置为 `egress` 表示出方向
- **ethertype**：网络类型，设置为 `IPv4`
- **remote_ip_prefix**：远端IP网段，设置为 `0.0.0.0/0` 表示允许访问任意公网地址
- **priority**：规则优先级，设置为 `1`

### 8. 创建ECS实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源
variable "ecs_instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "ecs_image_id" {
  description = "The ID of the ECS instance image"
  type        = string
  default     = ""
  nullable    = false
}

variable "ecs_system_disk_type" {
  description = "The type of the ECS instance system disk"
  type        = string
  default     = "SSD"
}

variable "ecs_system_disk_size" {
  description = "The size of the ECS instance system disk in GB"
  type        = number
  default     = 40
}

variable "ecs_admin_password" {
  description = "The password of the ECS instance administrator"
  type        = string
  sensitive   = true
}

variable "ecs_instance_tags" {
  description = "The tags of the ECS instance"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)
  flavor_id          = var.ecs_flavor_id != "" ? var.ecs_flavor_id : try(data.huaweicloud_compute_flavors.test.flavors[0].id, null)
  image_id           = var.ecs_image_id != "" ? var.ecs_image_id : try(data.huaweicloud_images_images.test.images[0].id, null)
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
- **name**：ECS实例名称，通过引用输入变量 ecs_instance_name 进行赋值
- **availability_zone**：ECS实例所在可用区，取可用区列表数据源的第一个可用区
- **flavor_id**：ECS实例规格ID，优先使用输入变量 ecs_flavor_id，为空时取计算规格列表数据源的第一个规格
- **image_id**：ECS实例镜像ID，优先使用输入变量 ecs_image_id，为空时取镜像列表数据源的第一个镜像
- **security_group_ids**：ECS实例绑定的安全组ID列表，引用 `huaweicloud_networking_secgroup.test.id`
- **system_disk_type**：系统盘类型，通过引用输入变量 ecs_system_disk_type 进行赋值
- **system_disk_size**：系统盘大小（GB），通过引用输入变量 ecs_system_disk_size 进行赋值
- **uuid**：ECS实例所属子网ID，引用 `huaweicloud_vpc_subnet.test.id`
- **admin_pass**：ECS实例管理员密码，通过引用输入变量 ecs_admin_password 进行赋值
- **tags**：ECS实例标签，通过引用输入变量 ecs_instance_tags 进行赋值

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC配置
vpc_name = "terraform-test-vpc"
vpc_cidr = "192.168.0.0/16"

# 子网配置
subnet_name       = "terraform-test-subnet"
subnet_cidr       = "192.168.1.0/24"
subnet_gateway_ip = "192.168.1.1"

# NAT网关配置
nat_gateway_name    = "terraform-test-nat-gateway"
gateway_spec        = "1"
gateway_description = "NAT gateway for terraform test"

# EIP配置
eip_bandwidth_name        = "terraform-test-eip-bandwidth"
eip_bandwidth_size        = 5
eip_bandwidth_share_type  = "PER"
eip_bandwidth_charge_mode = "traffic"

# SNAT规则配置（VPC场景）
snat_source_type = 0
snat_cidr        = "192.168.0.0/16"
snat_description = "SNAT rule for terraform test"

# ECS实例配置
ecs_instance_name           = "terraform-test-ecs"
ecs_flavor_id               = ""
ecs_flavor_performance_type = "normal"
ecs_flavor_cpu_core_count   = 2
ecs_flavor_memory_size      = 4
ecs_image_id                = ""
ecs_image_visibility        = "public"
ecs_image_os                = "Ubuntu"
ecs_security_group_name     = "terraform-test-ecs-sg"
ecs_admin_password          = "YourPassword123!"
ecs_system_disk_type        = "GPSSD"
ecs_system_disk_size        = 40
ecs_instance_tags = {
  Environment = "test"
  Project     = "terraform-example"
}
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建SNAT规则
4. 运行 `terraform show` 查看已创建的SNAT规则

## 参考信息

- [华为云NAT网关产品文档](https://support.huaweicloud.com/natgateway/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [NAT网关SNAT规则最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/nat/snat-basic)
