# 部署绑定网络接口的实例

## 应用场景

弹性云服务器（ECS）是华为云提供的可弹性伸缩的计算服务，当业务需要多网卡、多子网隔离或独立安全策略时，可以为ECS实例绑定额外的网络接口。通过网络接口绑定，实例可以在不同子网中同时通信，满足高可用、流量隔离和灵活组网等场景需求。

本最佳实践将介绍如何使用Terraform自动化部署一个绑定网络接口的ECS实例，包括可用区、规格和镜像的查询，VPC与子网的创建，安全组配置，实例创建以及网络接口的绑定。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS镜像列表（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [弹性云服务器（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [云服务器网卡挂载（huaweicloud_compute_interface_attach）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_interface_attach)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_compute_instance

data.huaweicloud_compute_flavors
    └── huaweicloud_compute_instance

data.huaweicloud_images_images
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_compute_instance
        └── huaweicloud_compute_interface_attach

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance

huaweicloud_compute_instance
    └── huaweicloud_compute_interface_attach
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区、ECS规格和镜像信息

在TF文件（如main.tf）中添加以下脚本以查询部署ECS实例所需的可用区、规格和镜像信息：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区、ECS规格和镜像信息
variable "availability_zone" {
  description = "The availability zone to which the ECS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_id" {
  description = "The flavor ID of the ECS instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "instance_cpu_core_count" {
  description = "The number of CPU cores of the ECS instance"
  type        = number
  default     = 2
}

variable "instance_memory_size" {
  description = "The memory size in GB of the ECS instance"
  type        = number
  default     = 4
}

variable "instance_image_id" {
  description = "The image ID of the ECS instance"
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
  description = "The operating system of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  performance_type  = var.instance_performance_type
  cpu_core_count    = var.instance_cpu_core_count
  memory_size       = var.instance_memory_size
}

data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}
```

**参数说明**：
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，当为空时自动查询当前region下的可用区列表
- **instance_flavor_id**：通过引用输入变量 instance_flavor_id 进行赋值，当为空时根据规格性能类型、CPU核数和内存大小查询规格列表
- **instance_performance_type**：通过引用输入变量 instance_performance_type 进行赋值，用于筛选规格的性能类型
- **instance_cpu_core_count**：通过引用输入变量 instance_cpu_core_count 进行赋值，用于筛选规格的CPU核数
- **instance_memory_size**：通过引用输入变量 instance_memory_size 进行赋值，用于筛选规格的内存大小
- **instance_image_id**：通过引用输入变量 instance_image_id 进行赋值，当为空时根据规格、可见性和操作系统查询镜像列表
- **instance_image_visibility**：通过引用输入变量 instance_image_visibility 进行赋值，用于筛选镜像的可见性
- **instance_image_os**：通过引用输入变量 instance_image_os 进行赋值，用于筛选镜像的操作系统

### 3. 创建VPC和子网

在TF文件（如main.tf）中添加以下脚本以创建VPC和子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC和子网
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

variable "subnet_configurations" {
  description = "The list of subnet configurations for ECS instance."
  type        = list(object({
    subnet_name       = string
    subnet_cidr       = optional(string)
    subnet_gateway_ip = optional(string)
  }))
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  count = length(var.subnet_configurations)

  vpc_id     = huaweicloud_vpc.test.id
  name       = lookup(var.subnet_configurations[count.index], "subnet_name", null)
  cidr       = try(coalesce(lookup(var.subnet_configurations[count.index], "subnet_cidr", null), cidrsubnet(huaweicloud_vpc.test.cidr, 8, count.index)), null)
  gateway_ip = try(coalesce(lookup(var.subnet_configurations[count.index], "subnet_gateway_ip", null), cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, count.index), 1)), null)
}
```

**参数说明**：
- **name**：通过引用输入变量 vpc_name 进行赋值，用于指定VPC名称
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值，用于指定VPC的网段
- **vpc_id**：通过引用VPC资源的ID进行赋值
- **subnet_name**：通过引用输入变量 subnet_configurations 中的 subnet_name 进行赋值，用于指定子网名称
- **subnet_cidr**：通过引用输入变量 subnet_configurations 中的 subnet_cidr 进行赋值，未指定时根据VPC网段自动计算
- **subnet_gateway_ip**：通过引用输入变量 subnet_configurations 中的 subnet_gateway_ip 进行赋值，未指定时根据子网网段自动计算

### 4. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值，用于指定安全组名称
- **delete_default_rules**：设置为true以删除安全组默认规则

### 5. 创建ECS实例

在TF文件（如main.tf）中添加以下脚本以创建ECS实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例
variable "instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "instance_admin_password" {
  description = "The login password of the ECS instance"
  type        = string
  sensitive   = true
}

resource "huaweicloud_compute_instance" "test" {
  name               = var.instance_name
  image_id           = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
  flavor_id          = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  availability_zone  = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  admin_pass         = var.instance_admin_password

  network {
    uuid = try(huaweicloud_vpc_subnet.test[0].id, null)
  }

  # When using `huaweicloud_compute_interface_attach`, if the security group is not specified, the default security group will be automatically added to the ECS instance.
  lifecycle {
    ignore_changes = [
      security_group_ids
    ]
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 instance_name 进行赋值，用于指定ECS实例名称
- **image_id**：通过引用输入变量 instance_image_id 进行赋值，未指定时使用查询到的镜像ID
- **flavor_id**：通过引用输入变量 instance_flavor_id 进行赋值，未指定时使用查询到的规格ID
- **security_group_ids**：通过引用安全组资源的ID进行赋值
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，未指定时使用查询到的可用区名称
- **admin_pass**：通过引用输入变量 instance_admin_password 进行赋值，用于指定实例登录密码
- **network/uuid**：通过引用第一个子网资源的ID进行赋值，作为实例的主网卡网络

### 6. 绑定网络接口

在TF文件（如main.tf）中添加以下脚本以将网络接口绑定到ECS实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下绑定网络接口到ECS实例
variable "attached_network_id" {
  description = "The ID of the network to which the ECS instance to be attached"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.attached_network_id != "" || length(var.subnet_configurations) == 2
    error_message = "When attached_network_id is not provided, subnet_configurations must have exactly 2 elements."
  }
}

variable "attached_interface_fixed_ip" {
  description = "The fixed IP address of the ECS instance to be attached"
  type        = string
  default     = null
}

variable "attached_security_group_ids" {
  description = "The list of security group IDs of the ECS instance to be attached"
  type        = list(string)
  default     = null
}

resource "huaweicloud_compute_interface_attach" "test" {
  instance_id        = huaweicloud_compute_instance.test.id
  network_id         = var.attached_network_id != "" ? var.attached_network_id : try(huaweicloud_vpc_subnet.test[1].id, null)
  fixed_ip           = var.attached_interface_fixed_ip
  security_group_ids = var.attached_security_group_ids
}
```

**参数说明**：
- **instance_id**：通过引用ECS实例资源的ID进行赋值
- **network_id**：通过引用输入变量 attached_network_id 进行赋值，未指定时使用第二个子网资源的ID
- **fixed_ip**：通过引用输入变量 attached_interface_fixed_ip 进行赋值，用于指定网络接口的固定IP地址
- **security_group_ids**：通过引用输入变量 attached_security_group_ids 进行赋值，用于指定网络接口的安全组

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
vpc_name              = "tf_test_ecs_instace"
subnet_configurations = [
  {
    subnet_name = "tf_test_main"
  },
  {
    subnet_name = "tf_test_standby"
  },
]

security_group_name     = "tf_test_ecs_instace"
instance_name           = "tf_test_ecs_instace"
instance_admin_password = "YourPassword!"
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

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建绑定网络接口的ECS实例
4. 运行 `terraform show` 查看已创建的绑定网络接口的ECS实例

## 参考信息

- [华为云ECS产品文档](https://support.huaweicloud.com/ecs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ECS绑定网络接口最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ecs/attached-interface)
