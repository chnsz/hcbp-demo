# 部署跨区域IPv4网络

## 应用场景

弹性公网IP（Elastic IP，EIP）是华为云提供的一种可独立申请和绑定的公网IP地址资源，为云上资源提供访问公网和被公网访问的能力。当业务需要跨区域构建统一的公网访问入口时，可以借助全域弹性公网IP（Global EIP，G-EIP）与全球互联网带宽、全球连接带宽，将不同区域的云资源统一接入同一份公网出口，实现跨区域网络的互联互通。

本最佳实践将介绍如何使用Terraform自动化部署跨区域IPv4网络，包括VPC与子网创建、安全组及安全组规则配置、ECS实例创建、VPC互联网网关创建、全域弹性公网IP池查询、全球互联网带宽创建、全域弹性公网IP创建，以及将全域弹性公网IP与ECS实例、VPC互联网网关和全球连接带宽进行关联。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像列表（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)
- [全域弹性公网IP池列表（data.huaweicloud_global_eip_pools）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/global_eip_pools)
- [IAM项目列表（data.huaweicloud_identity_projects）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [弹性云服务器（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [VPC互联网网关（huaweicloud_vpc_internet_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_internet_gateway)
- [全球互联网带宽（huaweicloud_global_internet_bandwidth）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/global_internet_bandwidth)
- [全域弹性公网IP（huaweicloud_global_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/global_eip)
- [全域弹性公网IP关联（huaweicloud_global_eip_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/global_eip_associate)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
data.huaweicloud_compute_flavors.test
data.huaweicloud_images_images.test
data.huaweicloud_global_eip_pools.test
data.huaweicloud_identity_projects.test
    └── huaweicloud_vpc.test
        └── huaweicloud_vpc_subnet.test
            └── huaweicloud_networking_secgroup.test
                └── huaweicloud_networking_secgroup_rule.test
                    └── huaweicloud_compute_instance.test
                        └── huaweicloud_vpc_internet_gateway.test
                            └── huaweicloud_global_internet_bandwidth.test
                                └── huaweicloud_global_eip.test
                                    └── huaweicloud_global_eip_associate.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询当前区域下的可用区列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
data "huaweicloud_availability_zones" "test" {}
```

### 3. 查询ECS规格列表

在TF文件（如main.tf）中添加以下脚本以查询ECS实例规格列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询ECS规格列表
variable "instance_flavor_id" {
  description = "The ID of the ECS instance flavor"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "The CPU core count of the ECS instance flavor"
  type        = number
  default     = 2
}

variable "instance_flavor_memory_size" {
  description = "The memory size of the ECS instance flavor"
  type        = number
  default     = 4
}

data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**参数说明**：
- **count**：当未指定实例规格ID时，才执行规格查询
- **availability_zone**：通过引用可用区列表数据源的第一个可用区进行赋值
- **performance_type**：通过引用输入变量 instance_flavor_performance_type 进行赋值
- **cpu_core_count**：通过引用输入变量 instance_flavor_cpu_core_count 进行赋值
- **memory_size**：通过引用输入变量 instance_flavor_memory_size 进行赋值

### 4. 查询镜像列表

在TF文件（如main.tf）中添加以下脚本以查询ECS实例镜像列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询镜像列表
variable "instance_image_id" {
  description = "The ID of the ECS instance image"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_image_visibility" {
  description = "The visibility of the ECS instance image"
  type        = string
  default     = "public"
}

variable "instance_image_os" {
  description = "The OS of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}
```

**参数说明**：
- **count**：当未指定镜像ID时，才执行镜像查询
- **flavor_id**：当未指定实例规格ID时，通过引用规格列表数据源的第一个规格ID进行赋值
- **visibility**：通过引用输入变量 instance_image_visibility 进行赋值
- **os**：通过引用输入变量 instance_image_os 进行赋值

### 5. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **name**：通过引用输入变量 vpc_name 进行赋值
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，未指定时使用默认企业项目

### 6. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 4, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 4, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：通过引用虚拟私有云的ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，未指定时基于VPC网段自动划分子网网段
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，未指定时基于子网网段自动计算网关地址

### 7. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值

### 8. 创建安全组规则

在TF文件（如main.tf）中添加以下脚本以创建安全组规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组规则
variable "security_group_rule_configurations" {
  description = "The list of security group rule configurations"

  type = list(object({
    direction        = optional(string, "ingress")
    ethertype        = optional(string, "IPv4")
    protocol         = optional(string, null)
    ports            = optional(string, null)
    remote_ip_prefix = optional(string, "0.0.0.0/0")
  }))

  nullable = false
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = length(var.security_group_rule_configurations)

  direction         = lookup(var.security_group_rule_configurations[count.index], "direction", "ingress")
  ethertype         = lookup(var.security_group_rule_configurations[count.index], "ethertype", "IPv4")
  protocol          = lookup(var.security_group_rule_configurations[count.index], "protocol", null)
  ports             = lookup(var.security_group_rule_configurations[count.index], "ports", null)
  remote_ip_prefix  = lookup(var.security_group_rule_configurations[count.index], "remote_ip_prefix", "0.0.0.0/0")
  security_group_id = huaweicloud_networking_secgroup.test.id
}
```

**参数说明**：
- **count**：通过引用输入变量 security_group_rule_configurations 的长度进行赋值
- **direction**：通过引用输入变量 security_group_rule_configurations 中的 direction 进行赋值
- **ethertype**：通过引用输入变量 security_group_rule_configurations 中的 ethertype 进行赋值
- **protocol**：通过引用输入变量 security_group_rule_configurations 中的 protocol 进行赋值
- **ports**：通过引用输入变量 security_group_rule_configurations 中的 ports 进行赋值
- **remote_ip_prefix**：通过引用输入变量 security_group_rule_configurations 中的 remote_ip_prefix 进行赋值
- **security_group_id**：通过引用安全组的ID进行赋值

### 9. 创建弹性云服务器

在TF文件（如main.tf）中添加以下脚本以创建弹性云服务器：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性云服务器
variable "instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "instance_administrator_password" {
  description = "The administrator password of the ECS instance"
  type        = string
  sensitive   = true
}

resource "huaweicloud_compute_instance" "test" {
  name               = var.instance_name
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, "") : var.instance_flavor_id
  image_id           = var.instance_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, "") : var.instance_image_id
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  admin_pass         = var.instance_administrator_password

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  depends_on = [huaweicloud_networking_secgroup_rule.test]

  lifecycle {
    ignore_changes = [
      availability_zone,
      flavor_id,
      image_id,
      admin_pass,
    ]
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 instance_name 进行赋值
- **availability_zone**：通过引用可用区列表数据源的第一个可用区进行赋值
- **flavor_id**：当未指定实例规格ID时，通过引用规格列表数据源的第一个规格ID进行赋值
- **image_id**：当未指定镜像ID时，通过引用镜像列表数据源的第一个镜像ID进行赋值
- **security_group_ids**：通过引用安全组的ID进行赋值
- **admin_pass**：通过引用输入变量 instance_administrator_password 进行赋值
- **network.uuid**：通过引用子网的ID进行赋值

### 10. 创建VPC互联网网关

在TF文件（如main.tf）中添加以下脚本以创建VPC互联网网关：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC互联网网关
variable "internet_gateway_name" {
  description = "The name of the VPC internet gateway"
  type        = string
}

variable "internet_gateway_add_route" {
  description = "Whether to add route to the internet gateway"
  type        = bool
  default     = true
}

resource "huaweicloud_vpc_internet_gateway" "test" {
  vpc_id    = huaweicloud_vpc.test.id
  subnet_id = huaweicloud_vpc_subnet.test.id
  name      = var.internet_gateway_name
  add_route = var.internet_gateway_add_route
}
```

**参数说明**：
- **vpc_id**：通过引用虚拟私有云的ID进行赋值
- **subnet_id**：通过引用子网的ID进行赋值
- **name**：通过引用输入变量 internet_gateway_name 进行赋值
- **add_route**：通过引用输入变量 internet_gateway_add_route 进行赋值

### 11. 查询全域弹性公网IP池列表

在TF文件（如main.tf）中添加以下脚本以查询全域弹性公网IP池列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询全域弹性公网IP池列表
variable "global_eip_access_site" {
  description = "The access site used to filter the global EIP pool"
  type        = string
  default     = "cn-north-beijing"
  nullable    = false
}

variable "global_eip_ip_version" {
  description = "The IP version of the global EIP"
  type        = string
  default     = "4"
}

data "huaweicloud_global_eip_pools" "test" {
  access_site = var.global_eip_access_site != "" ? var.global_eip_access_site : null
  ip_version  = var.global_eip_ip_version
}
```

**参数说明**：
- **access_site**：通过引用输入变量 global_eip_access_site 进行赋值
- **ip_version**：通过引用输入变量 global_eip_ip_version 进行赋值

### 12. 创建全球互联网带宽

在TF文件（如main.tf）中添加以下脚本以创建全球互联网带宽：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全球互联网带宽
variable "internet_bandwidth_charge_mode" {
  description = "The charge mode of the global internet bandwidth"
  type        = string
  default     = "95peak_guar"
}

variable "internet_bandwidth_size" {
  description = "The size of the global internet bandwidth in Mbit/s"
  type        = number
  default     = 300
}

variable "internet_bandwidth_name" {
  description = "The name of the global internet bandwidth"
  type        = string
  default     = null
}

variable "internet_bandwidth_ingress_size" {
  description = "The ingress size of the global internet bandwidth in Mbit/s"
  type        = number
  default     = null
}

variable "internet_bandwidth_tags" {
  description = "The tags of the internet bandwidth"
  type        = map(string)
  default     = null
}

resource "huaweicloud_global_internet_bandwidth" "test" {
  access_site           = try(data.huaweicloud_global_eip_pools.test.geip_pools[0].access_site, null)
  charge_mode           = var.internet_bandwidth_charge_mode
  size                  = var.internet_bandwidth_size
  isp                   = try(data.huaweicloud_global_eip_pools.test.geip_pools[0].isp, null)
  name                  = var.internet_bandwidth_name
  ingress_size          = var.internet_bandwidth_ingress_size
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  tags                  = var.internet_bandwidth_tags
}
```

**参数说明**：
- **access_site**：通过引用全域弹性公网IP池列表数据源的接入站点进行赋值
- **charge_mode**：通过引用输入变量 internet_bandwidth_charge_mode 进行赋值
- **size**：通过引用输入变量 internet_bandwidth_size 进行赋值
- **isp**：通过引用全域弹性公网IP池列表数据源的运营商信息进行赋值
- **name**：通过引用输入变量 internet_bandwidth_name 进行赋值
- **ingress_size**：通过引用输入变量 internet_bandwidth_ingress_size 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，未指定时使用默认企业项目
- **tags**：通过引用输入变量 internet_bandwidth_tags 进行赋值

### 13. 创建全域弹性公网IP

在TF文件（如main.tf）中添加以下脚本以创建全域弹性公网IP：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全域弹性公网IP
variable "global_eip_name" {
  description = "The name of the global EIP"
  type        = string
}

variable "global_eip_description" {
  description = "The description of the global EIP"
  type        = string
  default     = ""
}

variable "global_eip_tags" {
  description = "The tags of the global EIP"
  type        = map(string)
  default     = null
}

resource "huaweicloud_global_eip" "test" {
  access_site           = huaweicloud_global_internet_bandwidth.test.access_site
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  geip_pool_name        = try(data.huaweicloud_global_eip_pools.test.geip_pools[0].name, null)
  internet_bandwidth_id = huaweicloud_global_internet_bandwidth.test.id
  name                  = var.global_eip_name
  description           = var.global_eip_description
  tags                  = var.global_eip_tags
}
```

**参数说明**：
- **access_site**：通过引用全球互联网带宽的接入站点进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，未指定时使用默认企业项目
- **geip_pool_name**：通过引用全域弹性公网IP池列表数据源的池名称进行赋值
- **internet_bandwidth_id**：通过引用全球互联网带宽的ID进行赋值
- **name**：通过引用输入变量 global_eip_name 进行赋值
- **description**：通过引用输入变量 global_eip_description 进行赋值
- **tags**：通过引用输入变量 global_eip_tags 进行赋值

### 14. 查询IAM项目列表

在TF文件（如main.tf）中添加以下脚本以查询ECS实例所在区域的IAM项目列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询IAM项目列表
data "huaweicloud_identity_projects" "test" {
  name = huaweicloud_compute_instance.test.region
}
```

**参数说明**：
- **name**：通过引用弹性云服务器所在区域进行赋值

### 15. 创建全域弹性公网IP关联

在TF文件（如main.tf）中添加以下脚本以将全域弹性公网IP与ECS实例、VPC互联网网关和全球连接带宽进行关联：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全域弹性公网IP关联
variable "gc_bandwidth_name" {
  description = "The name of the global connection bandwidth"
  type        = string
}

variable "gc_bandwidth_charge_mode" {
  description = "The charge mode of the global connection bandwidth"
  type        = string
  default     = "95"
}

variable "gc_bandwidth_size" {
  description = "The size of the global connection bandwidth in Mbit/s"
  type        = number
  default     = 100
}

resource "huaweicloud_global_eip_associate" "test" {
  global_eip_id  = huaweicloud_global_eip.test.id
  is_reserve_gcb = false

  associate_instance {
    region        = huaweicloud_compute_instance.test.region
    project_id    = try(data.huaweicloud_identity_projects.test.projects[0].id, null)
    instance_type = "ECS"
    instance_id   = huaweicloud_compute_instance.test.id
  }

  gc_bandwidth {
    name                  = var.gc_bandwidth_name
    charge_mode           = var.gc_bandwidth_charge_mode
    size                  = var.gc_bandwidth_size
    enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  }

  depends_on = [huaweicloud_vpc_internet_gateway.test]
}
```

**参数说明**：
- **global_eip_id**：通过引用全域弹性公网IP的ID进行赋值
- **is_reserve_gcb**：是否保留全球连接带宽，此处设置为false
- **associate_instance.region**：通过引用弹性云服务器所在区域进行赋值
- **associate_instance.project_id**：通过引用IAM项目列表数据源的第一个项目ID进行赋值
- **associate_instance.instance_type**：关联的实例类型，此处设置为ECS
- **associate_instance.instance_id**：通过引用弹性云服务器的ID进行赋值
- **gc_bandwidth.name**：通过引用输入变量 gc_bandwidth_name 进行赋值
- **gc_bandwidth.charge_mode**：通过引用输入变量 gc_bandwidth_charge_mode 进行赋值
- **gc_bandwidth.size**：通过引用输入变量 gc_bandwidth_size 进行赋值
- **gc_bandwidth.enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，未指定时使用默认企业项目

### 16. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"

security_group_rule_configurations = [
  {
    direction        = "ingress"
    ethertype        = "IPv4"
    protocol         = "icmp"
    remote_ip_prefix = "0.0.0.0/0"
  },
  {
    direction        = "ingress"
    ethertype        = "IPv4"
    protocol         = "tcp"
    ports            = "22,3389"
    remote_ip_prefix = "10.1.0.7/32"
  },
  {
    direction        = "egress"
    ethertype        = "IPv4"
    remote_ip_prefix = "0.0.0.0/0"
  },
]

instance_name                   = "tf_test_ecs"
internet_gateway_name           = "tf_test_igw"
global_eip_name                 = "tf_test_geip"
internet_bandwidth_name         = "tf_test_internet_bandwidth"
gc_bandwidth_name               = "tf_test_gc_bandwidth"
instance_administrator_password = "YourPassword@123"
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

### 17. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建跨区域IPv4网络
4. 运行 `terraform show` 查看已创建的跨区域IPv4网络

## 参考信息

- [华为云弹性公网IP产品文档](https://support.huaweicloud.com/eip/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EIP跨区域IPv4网络最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eip/geip-ipv4-cross-region)
