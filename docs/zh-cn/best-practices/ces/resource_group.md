# 部署资源组

## 应用场景

云监控服务（Cloud Eye Service，CES）资源组是CES服务提供的资源分组管理功能，用于将多个云资源按照业务需求进行分组管理。通过配置资源组，您可以将相关的云资源（如ECS实例、RDS实例等）组织在一起，统一进行监控、告警和运维管理，提高资源管理的效率和便捷性。通过Terraform自动化创建CES资源组，可以确保资源分组配置的规范化和一致性，简化运维管理。本最佳实践将介绍如何使用Terraform自动化创建CES资源组。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 数据源

- [可用区数据源（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格数据源（huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [CES资源组资源（huaweicloud_ces_resource_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_resource_group)

## 资源依赖关系

本最佳实践中，资源之间存在以下依赖关系：

1. **CES资源组**依赖**ECS实例**等云资源，需要先创建这些资源才能将其添加到资源组中
2. **ECS实例**依赖**VPC子网**和**安全组**，需要先创建网络和安全组
3. **VPC子网**依赖**VPC**，需要先创建VPC
4. **ECS实例**依赖**可用区数据源**和**ECS规格数据源**，用于获取可用区和规格信息

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询数据源

在TF文件（如main.tf）中添加以下脚本以查询可用区和ECS规格信息：

```hcl
# 查询可用区信息
data "huaweicloud_availability_zones" "test" {}

# 查询ECS规格信息
data "huaweicloud_compute_flavors" "test" {
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.ecs_flavor_performance_type
  cpu_core_count    = var.ecs_flavor_cpu_core_count
  memory_size       = var.ecs_flavor_memory_size
}
```

**参数说明**：
- **availability_zone**：可用区名称，通过引用可用区数据源获取
- **performance_type**：ECS规格性能类型，通过引用输入变量ecs_flavor_performance_type进行赋值，默认值为"normal"
- **cpu_core_count**：ECS规格CPU核数，通过引用输入变量ecs_flavor_cpu_core_count进行赋值，默认值为2
- **memory_size**：ECS规格内存大小，通过引用输入变量ecs_flavor_memory_size进行赋值，默认值为4

### 3. 创建基础资源

在TF文件（如main.tf）中添加以下脚本以创建VPC、子网、安全组和ECS实例：

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
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
}

variable "security_group_name" {
  description = "The security group name"
  type        = string
  default     = "sg-dnat-backend"
}

variable "ecs_instance_name" {
  description = "The name of the ECS instance"
  type        = string
  default     = ""
}

variable "ecs_image_id" {
  description = "The ID of the ECS image"
  type        = string
  default     = ""
}

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

# 创建ECS实例
resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  image_id           = var.ecs_image_id
  flavor_id          = try(data.huaweicloud_compute_flavors.test.ids[0], null)
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }
}
```

### 4. 创建CES资源组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CES资源组资源：

```hcl
variable "resource_group_name" {
  description = "The name of the resource group"
  type        = string
}

variable "resource_group_resources" {
  description = "The resource list of the CES resource group"
  type = list(object({
    namespace  = string
    dimensions = list(object({
      name  = string
      value = string
    }))
  }))
  default = []
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CES资源组资源
resource "huaweicloud_ces_resource_group" "test" {
  name = var.resource_group_name

  dynamic "resources" {
    for_each = var.resource_group_resources

    content {
      namespace = resources.value.namespace
      dimensions {
        name  = resources.value.dimensions[0].name
        value = resources.value.dimensions[0].value
      }
    }
  }
}
```

**参数说明**：
- **name**：资源组名称，通过引用输入变量resource_group_name进行赋值
- **resources**：资源列表，通过引用输入变量resource_group_resources进行赋值，可选参数，默认值为空列表，每个资源包含以下参数：
  - **namespace**：服务命名空间，如SYS.ECS、SYS.RDS等
  - **dimensions**：资源维度列表，每个维度包含以下字段：
    - **name**：维度名称，如instance_id、rds_instance_id等
    - **value**：维度值，如ECS实例ID、RDS实例ID等

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC和子网配置
vpc_name    = "tf_test_ces_resource_group"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_ces_resource_group"

# 安全组配置
security_group_name = "tf_test_ces_resource_group"

# ECS实例配置
ecs_instance_name = "tf_test_ces_resource_group"
ecs_image_id      = "your_image_id"

# 资源组配置
resource_group_name = "tf_test_ces_resource_group"

# 资源组资源配置（示例：添加ECS实例到资源组）
resource_group_resources = [
  {
    namespace = "SYS.ECS"
    dimensions = [
      {
        name  = "instance_id"
        value = huaweicloud_compute_instance.test.id
      }
    ]
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是`ecs_image_id`需要替换为实际的镜像ID
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="resource_group_name=my_group" -var="vpc_name=my_vpc"`
2. 环境变量：`export TF_VAR_resource_group_name=my_group` 和 `export TF_VAR_vpc_name=my_vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建CES资源组：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建资源组及相关资源
4. 运行 `terraform show` 查看已创建的资源组详情

> 注意：资源组创建后，可以将多个云资源添加到资源组中进行统一管理。资源组支持按命名空间和维度进行资源筛选，可以灵活组织和管理云资源。资源组创建后，可以在CES控制台中进行进一步的配置，如创建告警规则、查看监控数据等。

## 参考信息

- [华为云CES产品文档](https://support.huaweicloud.com/ces/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [资源组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ces/resource-group)
