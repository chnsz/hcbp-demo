# 部署基础实例

## 应用场景

数据库安全服务（Database Security Service，DBSS）提供数据库安全审计能力，采用旁路部署方式，支持对华为云上RDS以及ECS/BMS自建数据库进行审计。通过创建基础DBSS审计实例，可搭建数据库安全审计的基础环境，为后续添加待审计数据库、开展风险识别与合规审计奠定基础。本最佳实践将介绍如何使用Terraform自动化部署一个基础的DBSS实例，包括创建VPC、子网、安全组，以及配置DBSS实例规格、计费方式等参数。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DBSS规格列表查询数据源（data.huaweicloud_dbss_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dbss_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DBSS实例资源（huaweicloud_dbss_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dbss_instance)

### 资源/数据源依赖关系

```text
data.huaweicloud_availability_zones
    └── huaweicloud_dbss_instance

data.huaweicloud_dbss_flavors
    └── huaweicloud_dbss_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dbss_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dbss_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建DBSS实例：

```hcl
variable "availability_zone" {
  description = "The availability zone to which the DBSS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建DBSS实例
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署DBSS实例
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 `vpc_name` 进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量 `vpc_cidr` 进行赋值

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
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署DBSS实例
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，通过引用VPC资源的ID进行赋值
- **name**：子网名称，通过引用输入变量 `subnet_name` 进行赋值
- **cidr**：子网的CIDR网段，如果输入变量为空则根据VPC网段自动计算，否则使用输入变量的值
- **gateway_ip**：子网的网关IP地址，如果输入变量为空则自动计算，否则使用输入变量的值

### 5. 创建安全组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署DBSS实例
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量 `security_group_name` 进行赋值
- **delete_default_rules**：是否删除默认安全组规则，本实践中固定为 `true`

### 6. 通过数据源查询DBSS规格

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建DBSS实例：

```hcl
variable "instance_flavor_id" {
  description = "The flavor ID of the DBSS instance"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的DBSS规格信息，用于创建DBSS实例
data "huaweicloud_dbss_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行DBSS规格列表查询数据源，仅当 `var.instance_flavor_id` 为空时创建数据源（即执行规格列表查询）

### 7. 创建DBSS实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DBSS实例资源：

```hcl
variable "instance_name" {
  description = "The name of the DBSS instance"
  type        = string
}

variable "instance_spec_code" {
  description = "The spec code of the DBSS instance"
  type        = string
  default     = "dbss.bypassaudit.low"
}

variable "instance_description" {
  description = "The description of the DBSS instance"
  type        = string
  default     = ""
}

variable "instance_tags" {
  description = "The tags of the DBSS instance"
  type        = map(string)
  default     = {}
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = null
}

variable "charging_mode" {
  description = "The charging mode of the DBSS instance"
  type        = string
  default     = "prePaid"
}

variable "period_unit" {
  description = "The period unit of the DBSS instance"
  type        = string
  default     = "month"
}

variable "period" {
  description = "The period of the DBSS instance"
  type        = number
  default     = 1
}

variable "auto_renew" {
  description = "The auto renew of the DBSS instance"
  type        = string
  default     = "false"
}

locals {
  vpc_name      = var.vpc_name
  subnet_name   = var.subnet_name
  instance_name = var.instance_name

  product_spec_desc = jsonencode(
    {
      "specDesc" : {
        "zh-cn" : {
          "主机名称" : local.instance_name,
          "虚拟私有云" : local.vpc_name,
          "子网" : local.subnet_name
        },
        "en-us" : {
          "Instance Name" : local.instance_name,
          "VPC" : local.vpc_name,
          "Subnet" : local.subnet_name
        }
      }
    }
  )
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DBSS实例资源
resource "huaweicloud_dbss_instance" "test" {
  name                  = var.instance_name
  availability_zone     = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  flavor                = var.instance_flavor_id == "" ? try(data.huaweicloud_dbss_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  product_spec_desc     = local.product_spec_desc
  resource_spec_code    = var.instance_spec_code
  description           = var.instance_description
  tags                  = var.instance_tags
  enterprise_project_id = var.enterprise_project_id
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew
}
```

**参数说明**：
- **name**：DBSS实例名称，通过引用输入变量 `instance_name` 进行赋值
- **availability_zone**：DBSS实例所属可用区，当输入变量为空时根据可用区列表查询数据源的返回结果进行赋值，否则使用输入变量的值
- **vpc_id**：DBSS实例所属的VPC ID，通过引用VPC资源的ID进行赋值
- **subnet_id**：DBSS实例所属的子网ID，通过引用VPC子网资源的ID进行赋值
- **security_group_id**：DBSS实例所属的安全组ID，通过引用安全组资源的ID进行赋值
- **flavor**：DBSS实例规格ID，当输入变量为空时根据DBSS规格列表查询数据源的返回结果进行赋值，否则使用输入变量的值
- **product_spec_desc**：产品规格描述信息，通过本地变量 `product_spec_desc` 进行赋值
- **resource_spec_code**：DBSS实例规格编码，通过引用输入变量 `instance_spec_code` 进行赋值
- **description**：DBSS实例描述信息，通过引用输入变量 `instance_description` 进行赋值
- **tags**：DBSS实例标签，通过引用输入变量 `instance_tags` 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 `enterprise_project_id` 进行赋值
- **charging_mode**：DBSS实例计费模式，通过引用输入变量 `charging_mode` 进行赋值
- **period_unit**：DBSS实例订购周期单位，通过引用输入变量 `period_unit` 进行赋值
- **period**：DBSS实例订购周期，通过引用输入变量 `period` 进行赋值
- **auto_renew**：是否自动续费，通过引用输入变量 `auto_renew` 进行赋值

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 网络与安全组配置
vpc_name            = "tf_test_instance"
subnet_name         = "tf_test_instance"
security_group_name = "tf_test_instance"

# DBSS实例配置
instance_name = "tf_test_instance"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=tf_test_instance" -var="instance_name=tf_test_instance"`
2. 环境变量：`export TF_VAR_instance_name=tf_test_instance`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DBSS实例
4. 运行 `terraform show` 查看已创建的DBSS实例

## 参考信息

- [华为云DBSS产品文档](https://support.huaweicloud.com/dbss/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DBSS基础实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dbss/basic-instance)
