# 部署灾难恢复演练

## 应用场景

业务恢复服务（Business Recovery Service，BRS）是一种为弹性云服务器（Elastic Cloud Server，ECS）和云硬盘（Elastic Volume Service，EVS）等服务提供容灾的服务。通过主机层复制、数据冗余和缓存加速等多项技术，提供给用户高级别的数据可靠性以及业务连续性，称为业务恢复服务。

业务恢复服务有助于保护业务应用，将弹性云服务器的数据、配置信息复制到容灾站点，并允许业务应用在生产站点云服务器停机期间在容灾站点云服务器上启动并正常运行，从而提升业务连续性。

灾难恢复演练是容灾方案中非常重要的一环，通过定期的灾难恢复演练，您可以验证容灾方案的可行性，确保在真正发生故障时能够快速恢复业务。本最佳实践将介绍如何使用Terraform自动化部署一个完整的BRS灾难恢复演练环境，包括保护组、受保护实例和灾难恢复演练。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)
- [SDRS域查询数据源（data.huaweicloud_sdrs_domain）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/sdrs_domain)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [SDRS保护组资源（huaweicloud_sdrs_protection_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sdrs_protection_group)
- [SDRS受保护实例资源（huaweicloud_sdrs_protected_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sdrs_protected_instance)
- [SDRS灾难恢复演练资源（huaweicloud_sdrs_drill）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sdrs_drill)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── data.huaweicloud_images_images
            └── huaweicloud_compute_instance

data.huaweicloud_sdrs_domain
    └── huaweicloud_sdrs_protection_group
        └── huaweicloud_sdrs_protected_instance
            └── huaweicloud_sdrs_drill

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    │   └── huaweicloud_compute_instance
    └── huaweicloud_sdrs_protection_group

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../docs/introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询ECS实例资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例（包括用于筛选规格）：

```hcl
variable "availability_zone" {
  description = "ECS实例所属的可用区信息"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建ECS实例资源
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 无特殊参数，获取当前region下所有可用区信息

### 3. 通过数据源查询ECS实例规格信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例：

```hcl
variable "instance_flavor_id" {
  description = "ECS实例规格ID"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "ECS实例规格性能类型"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "ECS实例规格CPU核数"
  type        = number
  default     = 2
}

variable "instance_flavor_memory_size" {
  description = "ECS实例规格内存大小"
  type        = number
  default     = 4
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的ECS规格信息，用于创建ECS实例
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行ECS规格列表查询数据源，仅当 `var.instance_flavor_id` 为空时创建数据源（即执行ECS规格列表查询）
- **availability_zone**：ECS实例所在的可用区，根据可用区列表查询数据源的返回结果进行赋值
- **performance_type**：ECS实例规格的性能类型，通过引用输入变量instance_flavor_performance_type进行赋值
- **cpu_core_count**：ECS实例规格的CPU核数，通过引用输入变量instance_flavor_cpu_core_count进行赋值
- **memory_size**：ECS实例规格的内存大小，通过引用输入变量instance_flavor_memory_size进行赋值

### 4. 通过数据源查询ECS实例镜像信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例：

```hcl
variable "instance_image_id" {
  description = "ECS实例镜像ID"
  type        = string
  default     = ""
}

variable "instance_image_os_type" {
  description = "ECS实例镜像操作系统类型"
  type        = string
  default     = "Ubuntu"
}

variable "instance_image_visibility" {
  description = "ECS实例镜像可见性"
  type        = string
  default     = "public"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的镜像信息，用于创建ECS实例
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].ids[0], "") : var.instance_flavor_id
  os         = var.instance_image_os_type
  visibility = var.instance_image_visibility
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行镜像列表查询数据源，仅当 `var.instance_image_id` 为空时创建数据源（即执行镜像列表查询）
- **flavor_id**：ECS实例规格ID，根据ECS规格列表查询数据源的返回结果进行赋值
- **os**：ECS实例镜像的操作系统类型，通过引用输入变量instance_image_os_type进行赋值
- **visibility**：ECS实例镜像的可见性，通过引用输入变量instance_image_visibility进行赋值

### 5. 通过数据源查询SDRS域信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建SDRS保护组：

```hcl
variable "sdrs_domain_name" {
  description = "SDRS域名称"
  type        = string
  default     = null
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的SDRS域信息，用于创建SDRS保护组
data "huaweicloud_sdrs_domain" "test" {
  name = var.sdrs_domain_name
}
```

**参数说明**：
- **name**：SDRS域的名称，通过引用输入变量sdrs_domain_name进行赋值

### 6. 创建VPC网络

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC网络资源：

```hcl
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR块"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源
resource "huaweicloud_vpc" "test" {
  count = 2

  name = count.index == 0 ? var.vpc_name : "${var.vpc_name}-drill"
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **count**：资源的创建数，用于创建生产环境VPC和灾难恢复演练环境VPC
- **name**：VPC的名称，生产环境使用var.vpc_name，灾难恢复演练环境使用"${var.vpc_name}-drill"
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值

### 7. 创建VPC子网

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "子网名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网CIDR块"
  type        = string
  default     = "192.168.0.0/24"
}

variable "subnet_gateway_ip" {
  description = "子网网关IP"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  count = 2

  vpc_id            = huaweicloud_vpc.test[count.index].id
  name              = count.index == 0 ? var.subnet_name : "${var.subnet_name}-drill"
  cidr              = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test[count.index].cidr, 8, 0) : var.subnet_cidr
  gateway_ip        = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test[count.index].cidr, 8, 0), 1) : var.subnet_gateway_ip
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**参数说明**：
- **count**：资源的创建数，用于创建生产环境子网和灾难恢复演练环境子网
- **vpc_id**：子网所属的VPC ID，根据VPC资源的返回结果进行赋值
- **name**：子网的名称，生产环境使用var.subnet_name，灾难恢复演练环境使用"${var.subnet_name}-drill"
- **cidr**：子网的CIDR块，通过引用输入变量subnet_cidr进行赋值，如果为空则自动计算
- **gateway_ip**：子网的网关IP，通过引用输入变量subnet_gateway_ip进行赋值，如果为空则自动计算
- **availability_zone**：子网所在的可用区，根据可用区列表查询数据源的返回结果进行赋值

### 8. 创建安全组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "安全组名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组的名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认的安全组规则

### 9. 创建ECS实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ECS实例资源：

```hcl
variable "ecs_instance_name" {
  description = "ECS实例名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  availability_zone  = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, "") : var.instance_flavor_id
  image_id           = var.instance_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, "") : var.instance_image_id
  security_group_ids = [huaweicloud_networking_secgroup.test.id]

  network {
    uuid = huaweicloud_vpc_subnet.test[0].id
  }
}
```

**参数说明**：
- **name**：ECS实例的名称，通过引用输入变量ecs_instance_name进行赋值
- **availability_zone**：ECS实例所在的可用区，根据可用区列表查询数据源的返回结果进行赋值
- **flavor_id**：ECS实例所使用规格的ID，根据ECS规格列表查询数据源的返回结果进行赋值
- **image_id**：ECS实例所使用镜像的ID，根据IMS镜像列表查询数据源的返回结果进行赋值
- **security_group_ids**：ECS实例所关联的安全组ID列表，根据安全组资源的返回结果进行赋值
- **network**：ECS实例的网络配置，关联到生产环境的子网

### 10. 创建SDRS保护组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建SDRS保护组资源：

```hcl
variable "protection_group_name" {
  description = "保护组名称"
  type        = string
}

variable "source_availability_zone" {
  description = "保护组生产站点可用区"
  type        = string
  default     = ""

  validation {
    condition = (
      (var.source_availability_zone == "" && var.target_availability_zone == "") ||
      (var.source_availability_zone != "" && var.target_availability_zone != "")
    )
    error_message = "Both `source_availability_zone` and `target_availability_zone` must be set, or both must be empty"
  }
}

variable "target_availability_zone" {
  description = "保护组容灾站点可用区"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SDRS保护组资源
resource "huaweicloud_sdrs_protection_group" "test" {
  name                     = var.protection_group_name
  source_availability_zone = var.source_availability_zone != "" ? var.source_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  target_availability_zone = var.target_availability_zone != "" ? var.target_availability_zone : try(data.huaweicloud_availability_zones.test.names[1], null)
  domain_id                = data.huaweicloud_sdrs_domain.test.id
  source_vpc_id            = huaweicloud_vpc.test[0].id
}
```

**参数说明**：
- **name**：保护组的名称，通过引用输入变量protection_group_name进行赋值
- **source_availability_zone**：保护组生产站点可用区，通过引用输入变量source_availability_zone进行赋值，如果为空则使用第一个可用区
- **target_availability_zone**：保护组容灾站点可用区，通过引用输入变量target_availability_zone进行赋值，如果为空则使用第二个可用区
- **domain_id**：SDRS域ID，根据SDRS域查询数据源的返回结果进行赋值
- **source_vpc_id**：生产站点VPC ID，根据生产环境VPC资源的返回结果进行赋值

### 11. 创建SDRS受保护实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建SDRS受保护实例资源：

```hcl
variable "protected_instance_name" {
  description = "受保护实例名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SDRS受保护实例资源
resource "huaweicloud_sdrs_protected_instance" "test" {
  name                 = var.protected_instance_name
  group_id             = huaweicloud_sdrs_protection_group.test.id
  server_id            = huaweicloud_compute_instance.test.id
  delete_target_server = true
  delete_target_eip    = true
}
```

**参数说明**：
- **name**：受保护实例的名称，通过引用输入变量protected_instance_name进行赋值
- **group_id**：保护组ID，根据SDRS保护组资源的返回结果进行赋值
- **server_id**：ECS实例ID，根据ECS实例资源的返回结果进行赋值
- **delete_target_server**：是否删除目标服务器，设置为true表示删除目标服务器
- **delete_target_eip**：是否删除目标EIP，设置为true表示删除目标EIP

### 12. 创建SDRS灾难恢复演练

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建SDRS灾难恢复演练资源：

```hcl
variable "drill_name" {
  description = "灾难恢复演练名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SDRS灾难恢复演练资源
resource "huaweicloud_sdrs_drill" "test" {
  name         = var.drill_name
  group_id     = huaweicloud_sdrs_protection_group.test.id
  drill_vpc_id = huaweicloud_vpc.test[1].id

  depends_on = [
    huaweicloud_sdrs_protected_instance.test,
    huaweicloud_vpc_subnet.test[1],
  ]
}
```

**参数说明**：
- **name**：灾难恢复演练的名称，通过引用输入变量drill_name进行赋值
- **group_id**：保护组ID，根据SDRS保护组资源的返回结果进行赋值
- **drill_vpc_id**：灾难恢复演练VPC ID，根据灾难恢复演练环境VPC资源的返回结果进行赋值
- **depends_on**：显式依赖关系，确保受保护实例和灾难恢复演练子网创建完成后再创建灾难恢复演练

### 13. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 资源基本信息
vpc_name                = "tf_test_sdrs_protection_instance_vpc"
subnet_name             = "tf_test_sdrs_protection_instance_subnet"
security_group_name     = "tf_test_sdrs_protection_instance_secgroup"
ecs_instance_name       = "tf_test_sdrs_protection_instance_ecs_instance"
protection_group_name   = "tf_test_sdrs_protection_instance_group"
protected_instance_name = "tf_test_sdrs_protected_instance"
drill_name              = "tf_test_sdrs_drill"

# 网络配置
vpc_cidr     = "192.168.0.0/16"
subnet_cidr  = "192.168.0.0/24"

# 实例配置
instance_flavor_performance_type = "normal"
instance_flavor_cpu_core_count   = 2
instance_flavor_memory_size      = 4
instance_image_os_type           = "Ubuntu"
instance_image_visibility        = "public"

# 容灾配置
source_availability_zone = ""
target_availability_zone = ""
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 14. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建BRS灾难恢复演练环境
4. 运行 `terraform show` 查看已创建的BRS灾难恢复演练环境

## 参考信息

- [华为云BRS产品文档](https://support.huaweicloud.com/sdrs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [BRS灾难恢复演练最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sdrs/drill)
