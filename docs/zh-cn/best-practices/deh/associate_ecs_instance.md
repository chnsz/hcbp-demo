# 部署关联ECS实例

## 应用场景

专属主机（Dedicated Host，DEH）是华为云提供的物理服务器资源，用于满足对资源独享、安全合规等有特殊要求的业务场景。通过将ECS实例部署到专属主机上，您可以获得物理服务器的完全控制权，实现资源的物理隔离，满足合规性要求。通过Terraform自动化在专属主机上部署ECS实例，可以确保资源部署的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化在专属主机上部署ECS实例。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 数据源

- [可用区数据源（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [专属主机类型数据源（huaweicloud_deh_types）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/deh_types)
- [镜像数据源（huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [专属主机实例资源（huaweicloud_deh_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/deh_instance)
- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)

## 资源依赖关系

本最佳实践中，资源之间存在以下依赖关系：

1. **ECS实例**依赖**专属主机实例**，需要通过scheduler_hints将ECS实例关联到专属主机
2. **ECS实例**依赖**VPC子网**和**安全组**，需要先创建网络和安全组
3. **VPC子网**依赖**VPC**，需要先创建VPC
4. **专属主机实例**依赖**可用区数据源**和**专属主机类型数据源**，用于获取可用区和主机类型信息
5. **ECS实例**依赖**镜像数据源**，用于获取镜像信息

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询数据源

在TF文件（如main.tf）中添加以下脚本以查询可用区、专属主机类型和镜像信息：

```hcl
# 查询可用区信息
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

# 查询专属主机类型信息
data "huaweicloud_deh_types" "test" {
  count = var.deh_instance_host_type == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}

# 查询镜像信息
data "huaweicloud_images_images" "test" {
  count = var.ecs_instance_image_id == "" ? 1 : 0

  flavor_id  = var.ecs_instance_flavor_id != "" ? var.ecs_instance_flavor_id : try(huaweicloud_deh_instance.test.host_properties[0].available_instance_capacities[0].flavor, null)
  visibility = var.ecs_instance_image_visibility
  os         = var.ecs_instance_image_os
}
```

**参数说明**：
- **availability_zone**：可用区名称，当availability_zone变量为空时查询
- **host_type**：专属主机类型，当deh_instance_host_type变量为空时查询
- **flavor_id**：ECS规格ID，用于查询匹配的镜像
- **visibility**：镜像可见性，通过引用输入变量ecs_instance_image_visibility进行赋值，默认值为"public"
- **os**：镜像操作系统，通过引用输入变量ecs_instance_image_os进行赋值，默认值为"Ubuntu"

### 3. 创建专属主机实例资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建专属主机实例资源：

```hcl
variable "deh_instance_name" {
  description = "The name of the dedicated host instance"
  type        = string
}

variable "availability_zone" {
  description = "The availability zone where the resources will be created"
  type        = string
  default     = ""
}

variable "deh_instance_host_type" {
  description = "The host type of the dedicated host"
  type        = string
  default     = ""
}

variable "deh_instance_auto_placement" {
  description = "Whether to enable auto placement for the dedicated host"
  type        = string
  default     = "on"
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the dedicated host"
  type        = string
  default     = null
}

variable "deh_instance_charging_mode" {
  description = "The charging mode of the dedicated host"
  type        = string
  default     = "prePaid"
}

variable "deh_instance_period_unit" {
  description = "The unit of the billing period of the dedicated host"
  type        = string
  default     = "month"
}

variable "deh_instance_period" {
  description = "The billing period of the dedicated host"
  type        = string
  default     = "1"
}

variable "deh_instance_auto_renew" {
  description = "Whether to enable auto renew for the dedicated host"
  type        = string
  default     = "false"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建专属主机实例资源
resource "huaweicloud_deh_instance" "test" {
  name                  = var.deh_instance_name
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  host_type             = var.deh_instance_host_type != "" ? var.deh_instance_host_type : try(data.huaweicloud_deh_types.test[0].dedicated_host_types[0].host_type, null)
  auto_placement        = var.deh_instance_auto_placement
  enterprise_project_id = var.enterprise_project_id
  charging_mode         = var.deh_instance_charging_mode
  period_unit           = var.deh_instance_period_unit
  period                = var.deh_instance_period
  auto_renew            = var.deh_instance_auto_renew
}
```

**参数说明**：
- **name**：专属主机实例名称，通过引用输入变量deh_instance_name进行赋值
- **availability_zone**：可用区名称，通过引用输入变量availability_zone或可用区数据源进行赋值
- **host_type**：专属主机类型，通过引用输入变量deh_instance_host_type或专属主机类型数据源进行赋值
- **auto_placement**：是否启用自动放置，通过引用输入变量deh_instance_auto_placement进行赋值，默认值为"on"
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，可选参数，默认值为null
- **charging_mode**：计费模式，通过引用输入变量deh_instance_charging_mode进行赋值，默认值为"prePaid"（包年包月）
- **period_unit**：计费周期单位，通过引用输入变量deh_instance_period_unit进行赋值，默认值为"month"（月）
- **period**：计费周期，通过引用输入变量deh_instance_period进行赋值，默认值为"1"
- **auto_renew**：是否自动续费，通过引用输入变量deh_instance_auto_renew进行赋值，默认值为"false"

### 4. 创建基础网络资源

在TF文件（如main.tf）中添加以下脚本以创建VPC、子网和安全组：

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

variable "subnet_name" {
  description = "The name of the VPC subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the VPC subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the VPC subnet"
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
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}

# 创建安全组
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

### 5. 创建关联到专属主机的ECS实例资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建关联到专属主机的ECS实例资源：

```hcl
variable "ecs_instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "ecs_instance_image_id" {
  description = "The ID of the ECS instance image"
  type        = string
  default     = ""
}

variable "ecs_instance_flavor_id" {
  description = "The ID of the ECS instance flavor"
  type        = string
  default     = ""
}

variable "ecs_instance_image_visibility" {
  description = "The visibility of the ECS instance image"
  type        = string
  default     = "public"
}

variable "ecs_instance_image_os" {
  description = "The OS of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

variable "ecs_instance_admin_pass" {
  description = "The password of the ECS instance administrator"
  type        = string
  sensitive   = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建关联到专属主机的ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  availability_zone  = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  flavor_id          = var.ecs_instance_flavor_id != "" ? var.ecs_instance_flavor_id : try(huaweicloud_deh_instance.test.host_properties[0].available_instance_capacities[0].flavor, null)
  image_id           = var.ecs_instance_image_id != "" ? var.ecs_instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, "")
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  admin_pass         = var.ecs_instance_admin_pass

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  scheduler_hints {
    tenancy = "dedicated"
    deh_id  = huaweicloud_deh_instance.test.id
  }
}
```

**参数说明**：
- **name**：ECS实例名称，通过引用输入变量ecs_instance_name进行赋值
- **availability_zone**：可用区名称，通过引用输入变量availability_zone或可用区数据源进行赋值
- **flavor_id**：ECS规格ID，通过引用输入变量ecs_instance_flavor_id或专属主机实例的可用规格进行赋值
- **image_id**：镜像ID，通过引用输入变量ecs_instance_image_id或镜像数据源进行赋值
- **security_group_ids**：安全组ID列表，通过引用安全组资源进行赋值
- **admin_pass**：管理员密码，通过引用输入变量ecs_instance_admin_pass进行赋值
- **network.uuid**：网络子网ID，通过引用子网资源进行赋值
- **scheduler_hints.tenancy**：租户类型，设置为"dedicated"表示部署到专属主机
- **scheduler_hints.deh_id**：专属主机ID，通过引用专属主机实例资源进行赋值

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 专属主机实例配置
deh_instance_name       = "tf_test_deh_instance"

# VPC和子网配置
vpc_name                = "tf_test_vpc"
subnet_name             = "tf_test_subnet"

# 安全组配置
security_group_name     = "tf_test_security_group"

# ECS实例配置
ecs_instance_name       = "tf_test_ecs_instance"
ecs_instance_admin_pass = "YourPassword@12!"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是`ecs_instance_admin_pass`需要设置符合密码复杂度要求的密码
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="deh_instance_name=my_deh" -var="ecs_instance_name=my_ecs"`
2. 环境变量：`export TF_VAR_deh_instance_name=my_deh` 和 `export TF_VAR_ecs_instance_name=my_ecs`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建专属主机实例和关联的ECS实例：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建专属主机实例和ECS实例
4. 运行 `terraform show` 查看已创建的专属主机实例和ECS实例详情

> 注意：ECS实例的规格必须与专属主机类型匹配。当前仅支持在专属主机上部署按需（postPaid）计费的ECS实例。通过scheduler_hints配置可以将ECS实例关联到指定的专属主机，实现资源的物理隔离。

## 参考信息

- [华为云DEH产品文档](https://support.huaweicloud.com/deh/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [关联ECS实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/deh/associate-ecs-instance)
