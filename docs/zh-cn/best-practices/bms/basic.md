# 部署基础实例

## 应用场景

裸金属服务器（Bare Metal Server，BMS）是一种可随时获取的、高性能、高可用的物理服务器，提供专属的物理服务器资源，无虚拟化开销，满足高性能计算、数据库、大数据分析等对性能要求较高的业务场景。BMS实例提供物理服务器的完整控制权，支持自定义操作系统、网络配置和安全策略，适用于对性能、安全性和合规性有严格要求的应用场景。本最佳实践将介绍如何使用Terraform自动化部署一个基础的BMS实例，包括VPC、子网、安全组和密钥对的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [BMS规格列表查询数据源（data.huaweicloud_bms_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/bms_flavors)
- [IMS镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [KPS密钥对资源（huaweicloud_kps_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [BMS实例资源（huaweicloud_bms_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/bms_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_bms_flavors
    └── huaweicloud_bms_instance

data.huaweicloud_bms_flavors
    └── huaweicloud_bms_instance

data.huaweicloud_images_images
    └── huaweicloud_bms_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_bms_instance

huaweicloud_networking_secgroup
    └── huaweicloud_bms_instance

huaweicloud_kps_keypair
    └── huaweicloud_bms_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询BMS实例资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建BMS实例：

```hcl
variable "availability_zone" {
  description = "The availability zone to which the BMS instance belongs"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建BMS实例
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zone` 为空时创建数据源（即执行可用区列表查询）

### 3. 创建VPC资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署BMS实例
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"

### 4. 创建VPC子网资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署BMS实例
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源（huaweicloud_vpc.test）的ID
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR网段，通过引用输入变量subnet_cidr进行赋值，当值为空字符串时自动计算
- **gateway_ip**：子网的网关IP，通过引用输入变量subnet_gateway_ip进行赋值，当值为空字符串时自动计算

### 5. 创建安全组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署BMS实例
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true

### 6. 通过数据源查询BMS实例资源创建所需的规格

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建BMS实例：

```hcl
variable "instance_flavor_id" {
  description = "The flavor ID of the BMS instance"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的BMS规格信息，用于创建BMS实例
data "huaweicloud_bms_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  cpu_arch         = "aarch64"
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行BMS规格查询数据源，仅当 `var.instance_flavor_id` 为空时创建数据源（即执行规格查询）
- **cpu_arch**：CPU架构，设置为"aarch64"表示ARM架构
- **availability_zone**：可用区，当输入变量availability_zone不为空时使用该值，否则根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值

### 7. 通过数据源查询BMS实例资源创建所需的镜像

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建BMS实例：

```hcl
variable "instance_image_id" {
  description = "The image ID of the BMS instance"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的IMS镜像信息，用于创建BMS实例
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  os         = "Huawei Cloud EulerOS"
  image_type = "Ironic"
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行镜像查询数据源，仅当 `var.instance_image_id` 为空时创建数据源（即执行镜像查询）
- **os**：镜像的操作系统类型，设置为"Huawei Cloud EulerOS"操作系统
- **image_type**：镜像类型，设置为"Ironic"表示裸金属服务器镜像

### 8. 创建KPS密钥对资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建KPS密钥对资源：

```hcl
variable "keypair_name" {
  description = "The name of the KPS keypair"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建KPS密钥对资源，用于BMS实例的SSH登录
resource "huaweicloud_kps_keypair" "test" {
  name = var.keypair_name
}
```

**参数说明**：
- **name**：密钥对名称，通过引用输入变量keypair_name进行赋值

### 9. 创建BMS实例资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建BMS实例资源：

```hcl
variable "instance_name" {
  description = "The name of the BMS instance"
  type        = string
}

variable "instance_user_id" {
  description = "The user ID of the BMS instance"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the BMS instance belongs"
  type        = string
  default     = null
}

variable "instance_tags" {
  description = "The key/value pairs to associate with the BMS instance"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the BMS instance"
  type        = string
  default     = "prePaid"
}

variable "period_unit" {
  description = "The period unit of the BMS instance"
  type        = string
  default     = "month"
}

variable "period" {
  description = "The period of the BMS instance"
  type        = number
  default     = 1
}

variable "auto_renew" {
  description = "The auto renew of the BMS instance"
  type        = string
  default     = "false"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建BMS实例资源
resource "huaweicloud_bms_instance" "test" {
  name                  = var.instance_name
  user_id               = var.instance_user_id
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  vpc_id                = huaweicloud_vpc.test.id
  flavor_id             = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_bms_flavors.test[0].flavors[0].id, null)
  image_id              = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
  security_groups       = [huaweicloud_networking_secgroup.test.id]
  key_pair              = huaweicloud_kps_keypair.test.name
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.instance_tags
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew

  nics {
    subnet_id = huaweicloud_vpc_subnet.test.id
  }

  lifecycle {
    ignore_changes = [
      availability_zone,
      flavor_id,
      image_id,
    ]
  }
}
```

**参数说明**：
- **name**：BMS实例名称，通过引用输入变量instance_name进行赋值
- **user_id**：BMS实例用户ID，通过引用输入变量instance_user_id进行赋值
- **availability_zone**：可用区，当输入变量availability_zone不为空时使用该值，否则根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值
- **vpc_id**：VPC ID，引用前面创建的VPC资源（huaweicloud_vpc.test）的ID
- **flavor_id**：规格ID，当输入变量instance_flavor_id不为空时使用该值，否则根据BMS规格查询数据源（data.huaweicloud_bms_flavors）的返回结果进行赋值
- **image_id**：镜像ID，当输入变量instance_image_id不为空时使用该值，否则根据镜像查询数据源（data.huaweicloud_images_images）的返回结果进行赋值
- **security_groups**：安全组ID列表，引用前面创建的安全组资源（huaweicloud_networking_secgroup.test）的ID
- **key_pair**：密钥对名称，引用前面创建的KPS密钥对资源（huaweicloud_kps_keypair.test）的名称
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null
- **tags**：标签键值对，通过引用输入变量instance_tags进行赋值，默认值为空对象
- **charging_mode**：计费模式，通过引用输入变量charging_mode进行赋值，默认值为"prePaid"
- **period_unit**：计费周期单位，通过引用输入变量period_unit进行赋值，默认值为"month"
- **period**：计费周期，通过引用输入变量period进行赋值，默认值为1
- **auto_renew**：是否自动续费，通过引用输入变量auto_renew进行赋值，默认值为"false"
- **nics.subnet_id**：网卡所属的子网ID，引用前面创建的VPC子网资源（huaweicloud_vpc_subnet.test）的ID

### 10. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC配置
vpc_name    = "tf_test_bms_vpc"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_bms_subnet"

# 安全组配置
security_group_name = "tf_test_bms_security_group"

# KPS密钥对配置
keypair_name = "tf_test_kps_keypair"

# BMS实例配置
instance_name    = "tf_test_bms_instance"
instance_user_id = "your_user_id"
enterprise_project_id = "0"
instance_tags = {
  owner = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=test-vpc" -var="instance_name=test-instance"`
2. 环境变量：`export TF_VAR_vpc_name=test-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 11. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建BMS实例
4. 运行 `terraform show` 查看已创建的BMS实例详情

> 注意：BMS实例的创建大约需要30分钟，请耐心等待。

## 参考信息

- [华为云BMS产品文档](https://support.huaweicloud.com/bms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [基础实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/bms/bms-instance)
