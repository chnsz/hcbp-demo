# 部署云堡垒机单机实例

## 应用场景

云堡垒机（Cloud Bastion Host，CBH）是华为云提供的一款安全运维管理服务，为企业提供统一的安全运维入口。云堡垒机通过集中化的身份认证、权限管理、操作审计等功能，帮助企业建立安全、合规的运维管理体系。

云堡垒机支持多种协议（SSH、RDP、VNC等），提供细粒度的权限控制、完整的操作审计和实时监控能力。在创建之前，您需要根据实际的应用场景确认云堡垒机的规格类型、网络参数和安全组规则等配置。

本最佳实践将介绍如何使用Terraform自动化部署一个云堡垒机单机实例。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_cbh_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cbh_availability_zones)
- [云堡垒机规格列表查询数据源（data.huaweicloud_cbh_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cbh_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [云堡垒机单机实例资源（huaweicloud_cbh_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cbh_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_cbh_availability_zones
    └── huaweicloud_cbh_instance

data.huaweicloud_cbh_flavors
    └── huaweicloud_cbh_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_cbh_instance

huaweicloud_networking_secgroup
    └── huaweicloud_cbh_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../docs/introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询云堡垒机单机实例资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建云堡垒机单机实例：

```hcl
variable "availability_zone" {
  description = "云堡垒机单机实例所属的可用区信息"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建云堡垒机单机实例
data "huaweicloud_cbh_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zone` 为空时创建数据源（即执行可用区列表查询）

### 3. 通过数据源查询云堡垒机单机实例资源创建所需的规格信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建云堡垒机单机实例：

```hcl
variable "instance_flavor_id" {
  description = "云堡垒机单机实例的规格ID"
  type        = string
  default     = ""
}

variable "instance_flavor_type" {
  description = "云堡垒机单机实例的规格类型"
  type        = string
  default     = "basic"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的云堡垒机规格信息，用于创建云堡垒机单机实例
data "huaweicloud_cbh_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0
  type  = var.instance_flavor_type
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行规格列表查询数据源，仅当 `var.instance_flavor_id` 为空时创建数据源（即执行规格列表查询）
- **type**：云堡垒机单机实例的规格类型，通过引用输入变量instance_flavor_type进行赋值

### 4. 创建VPC

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "VPC的名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
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
- **name**：VPC的名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值

### 5. 创建VPC子网

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "子网的名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网的CIDR块"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "子网的网关IP"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，通过引用VPC资源（huaweicloud_vpc）的ID进行赋值
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR块，当subnet_cidr为空时自动计算，否则使用输入变量subnet_cidr的值
- **gateway_ip**：子网的网关IP，当subnet_gateway_ip为空时自动计算，否则使用输入变量subnet_gateway_ip的值

### 6. 创建安全组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "安全组的名称"
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
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 7. 创建云堡垒机单机实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云堡垒机单机实例资源：

```hcl
variable "instance_name" {
  description = "云堡垒机单机实例的名称"
  type        = string
}

variable "instance_password" {
  description = "云堡垒机单机实例的登录密码"
  type        = string
  sensitive   = true
}

variable "charging_mode" {
  description = "云堡垒机单机实例的计费模式"
  type        = string
  default     = "prePaid"
}

variable "period_unit" {
  description = "云堡垒机单机实例的计费周期单位"
  type        = string
  default     = "month"
}

variable "period" {
  description = "云堡垒机单机实例的计费周期"
  type        = number
  default     = 1
}

variable "auto_renew" {
  description = "云堡垒机单机实例是否开启自动续费"
  type        = string
  default     = "false"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云堡垒机单机实例资源
resource "huaweicloud_cbh_instance" "test" {
  name              = var.instance_name
  flavor_id         = var.instance_flavor_id == "" ? try(data.huaweicloud_cbh_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_cbh_availability_zones.test[0].availability_zones[0].name, null) : var.availability_zone
  password          = var.instance_password
  charging_mode     = var.charging_mode
  period_unit       = var.period_unit
  period            = var.period
  auto_renew        = var.auto_renew

  # If you want to change some of the following parameters, you need to remove the corresponding fields from "lifecycle.ignore_changes".
  lifecycle {
    ignore_changes = [
      flavor_id,
      availability_zone,
    ]
  }
}
```

**参数说明**：
- **name**：云堡垒机单机实例的名称，通过引用输入变量instance_name进行赋值
- **flavor_id**：云堡垒机单机实例的规格ID，当instance_flavor_id为空时根据规格列表查询数据源（data.huaweicloud_cbh_flavors）的返回结果进行赋值，否则使用输入变量instance_flavor_id的值
- **vpc_id**：云堡垒机单机实例所属的VPC ID，通过引用VPC资源（huaweicloud_vpc）的ID进行赋值
- **subnet_id**：云堡垒机单机实例所属的子网ID，通过引用VPC子网资源（huaweicloud_vpc_subnet）的ID进行赋值
- **security_group_id**：云堡垒机单机实例所属的安全组ID，通过引用安全组资源（huaweicloud_networking_secgroup）的ID进行赋值
- **availability_zone**：云堡垒机单机实例所在的可用区，当availability_zone为空时根据可用区列表查询数据源（data.huaweicloud_cbh_availability_zones）的返回结果进行赋值，否则使用输入变量availability_zone的值
- **password**：云堡垒机单机实例的登录密码，通过引用输入变量instance_password进行赋值
- **charging_mode**：云堡垒机单机实例的计费模式，通过引用输入变量charging_mode进行赋值
- **period_unit**：云堡垒机单机实例的计费周期单位，通过引用输入变量period_unit进行赋值
- **period**：云堡垒机单机实例的计费周期，通过引用输入变量period进行赋值
- **auto_renew**：云堡垒机单机实例是否开启自动续费，通过引用输入变量auto_renew进行赋值

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 资源基本信息
vpc_name            = "tf_test_cbh_instance_vpc"
subnet_name         = "tf_test_cbh_instance_subnet"
security_group_name = "tf_test_cbh_instance_security_group"
instance_name       = "tf_test_cbh_instance"
instance_password   = "YourCBHInstancePassword!"
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

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建云堡垒机单机实例
4. 运行 `terraform show` 查看已创建的云堡垒机单机实例

## 参考信息

- [华为云云堡垒机产品文档](https://support.huaweicloud.com/cbh/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [云堡垒机最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cbh/basic)
