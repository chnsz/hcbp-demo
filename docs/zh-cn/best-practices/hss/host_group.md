# 部署主机组

## 应用场景

主机安全服务（Host Security Service，HSS）是华为云提供的主机安全防护服务，提供资产管理、漏洞管理、入侵检测、基线检查等功能，帮助您全面保护云上主机的安全。通过创建HSS主机组，可以将多个主机进行分组管理，统一配置安全策略、执行安全检查和进行安全运维，提高主机安全管理的效率和便捷性。本最佳实践将介绍如何使用Terraform自动化部署HSS主机组，包括VPC、子网、安全组、ECS实例（带HSS agent）和HSS主机组的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区数据源（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格数据源（huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像数据源（huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [HSS主机组资源（huaweicloud_hss_host_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/hss_host_group)

### 资源/数据源依赖关系

```text
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── data.huaweicloud_images_images
            └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance
        └── huaweicloud_hss_host_group
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区数据源

在TF文件（如main.tf）中添加以下脚本以查询可用区信息：

```hcl
variable "availability_zone" {
  description = "The availability zone to which the ECS instance and network belong"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建ECS实例资源
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zone` 为空时创建数据源（即执行可用区列表查询）

### 3. 查询ECS规格数据源

在TF文件（如main.tf）中添加以下脚本以查询ECS规格信息：

```hcl
variable "instance_flavor_id" {
  description = "The flavor ID of the ECS instance"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "The number of the ECS instance flavor CPU cores"
  type        = number
  default     = 2
}

variable "instance_flavor_memory_size" {
  description = "The number of the ECS instance flavor memories"
  type        = number
  default     = 4
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的ECS规格信息，用于创建ECS实例资源
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行ECS规格列表查询数据源，仅当 `var.instance_flavor_id` 为空时创建数据源（即执行ECS规格列表查询）
- **availability_zone**：可用区，通过引用输入变量或可用区数据源进行赋值
- **performance_type**：性能类型，通过引用输入变量instance_flavor_performance_type进行赋值，默认值为"normal"
- **cpu_core_count**：CPU核数，通过引用输入变量instance_flavor_cpu_core_count进行赋值，默认值为2
- **memory_size**：内存大小（GB），通过引用输入变量instance_flavor_memory_size进行赋值，默认值为4

### 4. 查询镜像数据源

在TF文件（如main.tf）中添加以下脚本以查询镜像信息：

```hcl
variable "instance_image_id" {
  description = "The image ID of the ECS instance"
  type        = string
  default     = ""
}

variable "instance_image_os_type" {
  description = "The OS type of the ECS instance flavor"
  type        = string
  default     = "Ubuntu"
}

variable "instance_image_visibility" {
  description = "The visibility of the ECS instance flavor"
  type        = string
  default     = "public"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的镜像信息，用于创建ECS实例资源
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  os         = var.instance_image_os_type
  visibility = var.instance_image_visibility
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行镜像列表查询数据源，仅当 `var.instance_image_id` 为空时创建数据源（即执行镜像列表查询）
- **flavor_id**：规格ID，通过引用输入变量或ECS规格数据源进行赋值
- **os**：操作系统类型，通过引用输入变量instance_image_os_type进行赋值，默认值为"Ubuntu"
- **visibility**：镜像可见性，通过引用输入变量instance_image_visibility进行赋值，默认值为"public"

### 5. 创建VPC资源

在TF文件（如main.tf）中添加以下脚本以创建VPC：

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR地址块，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"

### 6. 创建VPC子网资源

在TF文件（如main.tf）中添加以下脚本以创建VPC子网：

```hcl
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "192.168.0.0/24"
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：VPC ID，通过引用VPC资源进行赋值
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR地址块，通过引用输入变量或自动计算进行赋值
- **gateway_ip**：子网网关IP，通过引用输入变量或自动计算进行赋值

### 7. 创建安全组资源

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 8. 创建ECS实例资源

在TF文件（如main.tf）中添加以下脚本以创建ECS实例：

```hcl
variable "ecs_instance_name" {
  type        = string
  description = "The name of the ECS instance"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  image_id           = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
  flavor_id          = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  availability_zone  = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  agent_list         = "hss"

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }
}
```

**参数说明**：
- **name**：ECS实例名称，通过引用输入变量ecs_instance_name进行赋值
- **image_id**：镜像ID，通过引用输入变量或镜像数据源进行赋值
- **flavor_id**：规格ID，通过引用输入变量或ECS规格数据源进行赋值
- **availability_zone**：可用区，通过引用输入变量或可用区数据源进行赋值
- **security_group_ids**：安全组ID列表，通过引用安全组资源进行赋值
- **agent_list**：Agent列表，设置为"hss"表示安装HSS agent
- **network.uuid**：网络子网ID，通过引用VPC子网资源进行赋值

> 注意：ECS实例需要配置`agent_list = "hss"`以安装HSS agent，这是创建HSS主机组的前提条件。ECS实例创建后需要等待HSS agent安装完成，才能将实例添加到HSS主机组中。

### 9. 创建HSS主机组资源

在TF文件（如main.tf）中添加以下脚本以创建HSS主机组：

```hcl
variable "host_group_name" {
  type        = string
  description = "The name of the HSS host group"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建HSS主机组资源
resource "huaweicloud_hss_host_group" "test" {
  name     = var.host_group_name
  host_ids = [huaweicloud_compute_instance.test.id]
}
```

**参数说明**：
- **name**：HSS主机组名称，通过引用输入变量host_group_name进行赋值
- **host_ids**：主机ID列表，通过引用ECS实例资源进行赋值

> 注意：HSS主机组需要包含已安装HSS agent的ECS实例。确保ECS实例已创建并完成HSS agent安装后，才能将实例添加到主机组中。

### 10. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC和子网配置
vpc_name            = "tf_test_hss_host_group_vpc"
vpc_cidr            = "192.168.0.0/16"
subnet_name         = "tf_test_hss_host_group_subnet"
subnet_cidr         = "192.168.0.0/24"

# 安全组配置
security_group_name = "tf_test_hss_host_group_secgroup"

# ECS实例配置
ecs_instance_name    = "tf_test_hss_host_group_ecs_instance"

# HSS主机组配置
host_group_name      = "tf_test_hss_host_group"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `availability_zone`可以设置可用区，如果为空则自动查询
   - `instance_flavor_id`可以设置ECS规格ID，如果为空则根据CPU和内存参数自动查询
   - `instance_image_id`可以设置镜像ID，如果为空则根据操作系统类型自动查询
   - `instance_flavor_performance_type`、`instance_flavor_cpu_core_count`、`instance_flavor_memory_size`可以设置ECS规格参数
   - `instance_image_os_type`、`instance_image_visibility`可以设置镜像参数
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="host_group_name=my-host-group"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc` 和 `export TF_VAR_host_group_name=my-host-group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。ECS实例需要配置`agent_list = "hss"`以安装HSS agent，这是创建HSS主机组的前提条件。

### 11. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建HSS主机组：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建VPC、子网、安全组、ECS实例和HSS主机组
4. 运行 `terraform show` 查看已创建的HSS主机组详情

> 注意：ECS实例创建后需要等待HSS agent安装完成，才能将实例添加到HSS主机组中。如果实例尚未完成HSS agent安装，主机组创建可能会失败。建议在创建主机组前确认ECS实例的HSS agent状态。

## 参考信息

- [华为云HSS产品文档](https://support.huaweicloud.com/hss/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [主机组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/hss/host-group)
