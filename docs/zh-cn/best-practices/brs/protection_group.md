# 部署保护组

## 应用场景

业务恢复服务（Business Recovery Service，BRS）是一种为弹性云服务器（Elastic Cloud Server，ECS）和云硬盘（Elastic Volume Service，EVS）等服务提供容灾的服务。通过主机层复制、数据冗余和缓存加速等多项技术，提供给用户高级别的数据可靠性以及业务连续性，称为业务恢复服务。

业务恢复服务有助于保护业务应用，将弹性云服务器的数据、配置信息复制到容灾站点，并允许业务应用在生产站点云服务器停机期间在容灾站点云服务器上启动并正常运行，从而提升业务连续性。

保护组是BRS容灾方案中的基础组件，用于定义容灾保护的范围和策略。通过创建保护组，您可以指定生产站点和容灾站点的可用区，建立容灾保护的基础架构。本最佳实践将介绍如何使用Terraform自动化部署一个BRS保护组环境。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [SDRS域查询数据源（data.huaweicloud_sdrs_domain）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/sdrs_domain)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [SDRS保护组资源（huaweicloud_sdrs_protection_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sdrs_protection_group)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_sdrs_protection_group

data.huaweicloud_sdrs_domain
    └── huaweicloud_sdrs_protection_group

huaweicloud_vpc
    └── huaweicloud_sdrs_protection_group
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../docs/introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC网络

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
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC的名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值

### 3. 通过数据源查询可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建SDRS保护组：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建SDRS保护组
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 无特殊参数，获取当前region下所有可用区信息

### 4. 通过数据源查询SDRS域信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建SDRS保护组：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的SDRS域信息，用于创建SDRS保护组
data "huaweicloud_sdrs_domain" "test" {}
```

**参数说明**：
- 无特殊参数，获取当前region下所有SDRS域信息

### 5. 创建SDRS保护组

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

variable "protection_group_dr_type" {
  description = "部署模型"
  type        = string
  default     = null
}

variable "protection_group_description" {
  description = "保护组的描述"
  type        = string
  default     = null
}

variable "protection_group_enable" {
  description = "是否启用保护组开始保护"
  type        = bool
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SDRS保护组资源
resource "huaweicloud_sdrs_protection_group" "test" {
  name                     = var.protection_group_name
  source_availability_zone = var.source_availability_zone != "" ? var.source_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  target_availability_zone = var.target_availability_zone != "" ? var.target_availability_zone : try(data.huaweicloud_availability_zones.test.names[1], null)
  domain_id                = data.huaweicloud_sdrs_domain.test.id
  source_vpc_id            = huaweicloud_vpc.test.id
  dr_type                  = var.protection_group_dr_type
  description              = var.protection_group_description
  enable                   = var.protection_group_enable
}
```

**参数说明**：
- **name**：保护组的名称，通过引用输入变量protection_group_name进行赋值
- **source_availability_zone**：保护组生产站点可用区，通过引用输入变量source_availability_zone进行赋值，如果为空则使用第一个可用区
- **target_availability_zone**：保护组容灾站点可用区，通过引用输入变量target_availability_zone进行赋值，如果为空则使用第二个可用区
- **domain_id**：SDRS域ID，根据SDRS域查询数据源的返回结果进行赋值
- **source_vpc_id**：生产站点VPC ID，根据VPC资源的返回结果进行赋值
- **dr_type**：部署模型，通过引用输入变量protection_group_dr_type进行赋值
- **description**：保护组的描述，通过引用输入变量protection_group_description进行赋值
- **enable**：是否启用保护组开始保护，通过引用输入变量protection_group_enable进行赋值

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 资源基本信息
vpc_name                     = "tf_test_sdrs_protection_group"
protection_group_name        = "tf_test_sdrs_protection_group"

# 网络配置
vpc_cidr = "192.168.0.0/16"

# 保护组配置
protection_group_description = "Created by terraform script"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="protection_group_name=my-group"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建BRS保护组
4. 运行 `terraform show` 查看已创建的BRS保护组

## 参考信息

- [华为云BRS产品文档](https://support.huaweicloud.com/sdrs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [BRS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sdrs)
