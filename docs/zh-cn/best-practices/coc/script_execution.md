# 部署脚本执行

## 应用场景

云运维中心（Cloud Operations Center, COC）是华为云提供的一站式运维管理平台，为企业提供统一的运维管理入口。云运维中心通过脚本管理、任务调度、监控告警等功能，帮助企业实现自动化运维和智能化管理，提升运维效率和质量。

脚本执行是COC服务的核心功能之一，支持在ECS实例上执行各种运维脚本，包括Shell脚本、Python脚本、PowerShell脚本等。通过脚本执行，可以实现自动化部署、系统配置、监控检查等运维任务。本最佳实践将介绍如何使用Terraform自动化部署COC脚本执行，包括创建ECS实例、安装UniAgent、创建脚本和执行脚本的完整流程。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [COC脚本资源（huaweicloud_coc_script）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/coc_script)
- [COC脚本执行资源（huaweicloud_coc_script_execute）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/coc_script_execute)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_compute_instance.test
            └── huaweicloud_coc_script_execute.test

data.huaweicloud_compute_flavors.test
    └── huaweicloud_compute_instance.test

data.huaweicloud_images_images.test
    └── huaweicloud_compute_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_compute_instance.test

huaweicloud_coc_script.test
    └── huaweicloud_coc_script_execute.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询ECS实例资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例：

```hcl
variable "availability_zone" {
  description = "ECS实例所属的可用区信息"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建ECS实例
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zone` 为空时创建数据源

### 3. 通过数据源查询ECS实例规格信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例：

```hcl
variable "instance_flavor_id" {
  description = "ECS实例规格ID，如果未指定，将使用符合条件的第一种可用规格"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "ECS实例规格的性能类型，当instance_flavor_id未指定时用于查询可用规格"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "ECS实例规格的CPU核数，当instance_flavor_id未指定时用于查询可用规格"
  type        = number
  default     = 4
}

variable "instance_flavor_memory_size" {
  description = "ECS实例规格的内存大小（GB），当instance_flavor_id未指定时用于查询可用规格"
  type        = number
  default     = 8
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下符合条件的所有ECS规格信息，用于创建ECS实例
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], "") : var.availability_zone
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行ECS规格列表查询数据源，仅当 `var.instance_flavor_id` 为空时创建数据源
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区列表查询数据源的第一个结果
- **performance_type**：性能类型，用于筛选ECS规格
- **cpu_core_count**：CPU核数，用于筛选ECS规格
- **memory_size**：内存大小，用于筛选ECS规格

### 4. 通过数据源查询ECS实例镜像信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例：

```hcl
variable "instance_image_id" {
  description = "ECS实例镜像ID，如果未指定，将使用符合条件的第一种可用镜像"
  type        = string
  default     = ""
}

variable "instance_image_os_type" {
  description = "ECS实例镜像的操作系统类型，当instance_image_id未指定时用于查询可用镜像"
  type        = string
  default     = "Ubuntu"
}

variable "instance_image_visibility" {
  description = "ECS实例镜像的可见性，当instance_image_id未指定时用于查询可用镜像"
  type        = string
  default     = "public"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下符合条件的所有镜像信息，用于创建ECS实例
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].ids[0], "") : var.instance_flavor_id
  os         = var.instance_image_os_type
  visibility = var.instance_image_visibility
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行镜像列表查询数据源，仅当 `var.instance_image_id` 为空时创建数据源
- **flavor_id**：规格ID，优先使用输入变量，如果为空则使用ECS规格列表查询数据源的第一个结果
- **os**：操作系统类型，用于筛选镜像
- **visibility**：可见性，用于筛选镜像

### 5. 创建VPC

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "VPC名称"
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
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值

### 6. 创建VPC子网

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "子网名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网的CIDR块，如果未指定，将在现有CIDR地址块内计算一个子网CIDR"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "子网的网关IP，如果未指定，将在现有CIDR地址块内计算一个网关IP"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  name              = var.subnet_name
  cidr              = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip        = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
}
```

**参数说明**：
- **vpc_id**：VPC ID，通过引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网CIDR块，优先使用输入变量，如果为空则通过cidrsubnet函数计算
- **gateway_ip**：网关IP，优先使用输入变量，如果为空则通过cidrhost函数计算
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区列表查询数据源的第一个结果

### 7. 创建安全组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "安全组名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值

> 注意：默认安全组规则不能删除，否则UniAgent安装会失败。

### 8. 创建ECS实例并安装UniAgent

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ECS实例资源：

```hcl
variable "instance_name" {
  description = "ECS实例名称"
  type        = string
}

variable "instance_user_data" {
  description = "ECS实例的用户数据，用于安装UniAgent"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  name               = var.instance_name
  availability_zone  = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], "") : var.availability_zone
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, "") : var.instance_flavor_id
  image_id           = var.instance_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, "") : var.instance_image_id
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  user_data          = var.instance_user_data

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }
}
```

**参数说明**：
- **name**：ECS实例名称，通过引用输入变量instance_name进行赋值
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区列表查询数据源的第一个结果
- **flavor_id**：规格ID，优先使用输入变量，如果为空则使用ECS规格列表查询数据源的第一个结果
- **image_id**：镜像ID，优先使用输入变量，如果为空则使用镜像列表查询数据源的第一个结果
- **security_group_ids**：安全组ID列表，通过引用安全组资源（huaweicloud_networking_secgroup.test）的ID进行赋值
- **user_data**：用户数据，通过引用输入变量instance_user_data进行赋值，用于安装UniAgent
- **network.uuid**：网络ID，通过引用VPC子网资源（huaweicloud_vpc_subnet.test）的ID进行赋值

### 9. 创建COC脚本

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建COC脚本资源：

```hcl
variable "script_name" {
  description = "脚本名称"
  type        = string
}

variable "script_description" {
  description = "脚本描述"
  type        = string
}

variable "script_risk_level" {
  description = "脚本风险级别"
  type        = string
}

variable "script_version" {
  description = "脚本版本"
  type        = string
}

variable "script_type" {
  description = "脚本类型"
  type        = string
}

variable "script_content" {
  description = "脚本内容"
  type        = string
}

variable "script_parameters" {
  description = "脚本参数列表"
  type = list(object({
    name        = string
    value       = string
    description = string
    sensitive   = optional(bool)
  }))

  nullable = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建COC脚本资源
resource "huaweicloud_coc_script" "test" {
  name        = var.script_name
  description = var.script_description
  risk_level  = var.script_risk_level
  version     = var.script_version
  type        = var.script_type
  content     = var.script_content

  dynamic "parameters" {
    for_each = var.script_parameters

    content {
      name        = parameters.value.name
      value       = parameters.value.value
      description = parameters.value.description
      sensitive   = parameters.value.sensitive
    }
  }
}
```

**参数说明**：
- **name**：脚本名称，通过引用输入变量script_name进行赋值
- **description**：脚本描述，通过引用输入变量script_description进行赋值
- **risk_level**：脚本风险级别，通过引用输入变量script_risk_level进行赋值
- **version**：脚本版本，通过引用输入变量script_version进行赋值
- **type**：脚本类型，通过引用输入变量script_type进行赋值
- **content**：脚本内容，通过引用输入变量script_content进行赋值
- **parameters.name**：参数名称，通过引用脚本参数列表中的name字段进行赋值
- **parameters.value**：参数值，通过引用脚本参数列表中的value字段进行赋值
- **parameters.description**：参数描述，通过引用脚本参数列表中的description字段进行赋值
- **parameters.sensitive**：参数是否敏感，通过引用脚本参数列表中的sensitive字段进行赋值

### 10. 创建COC脚本执行

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建COC脚本执行资源：

```hcl
variable "script_execute_timeout" {
  description = "脚本执行超时时间（秒）"
  type        = number
}

variable "script_execute_execute_user" {
  description = "脚本执行用户"
  type        = string
}

variable "script_execute_parameters" {
  description = "脚本执行参数列表"
  type = list(object({
    name  = string
    value = string
  }))

  nullable = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建COC脚本执行资源
resource "huaweicloud_coc_script_execute" "test" {
  script_id    = huaweicloud_coc_script.test.id
  instance_id  = huaweicloud_compute_instance.test.id
  timeout      = var.script_execute_timeout
  execute_user = var.script_execute_execute_user

  dynamic "parameters" {
    for_each = var.script_execute_parameters

    content {
      name  = parameters.value.name
      value = parameters.value.value
    }
  }
}
```

**参数说明**：
- **script_id**：脚本ID，通过引用COC脚本资源（huaweicloud_coc_script.test）的ID进行赋值
- **instance_id**：实例ID，通过引用ECS实例资源（huaweicloud_compute_instance.test）的ID进行赋值
- **timeout**：执行超时时间，通过引用输入变量script_execute_timeout进行赋值
- **execute_user**：执行用户，通过引用输入变量script_execute_execute_user进行赋值
- **parameters.name**：参数名称，通过引用脚本执行参数列表中的name字段进行赋值
- **parameters.value**：参数值，通过引用脚本执行参数列表中的value字段进行赋值

### 11. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 网络配置
vpc_name            = "tf_test_coc_script_execute_vpc"
subnet_name         = "tf_test_coc_script_execute_subnet"
security_group_name = "tf_test_coc_script_execute_secgroup"

# ECS实例配置
instance_name      = "tf_test_coc_script_execute"
instance_user_data = "your_user_data" # 请替换为您实际用于安装icagent的命令

# 脚本配置
script_name         = "tf_coc_script_execute"
script_description  = "Created by terraform script"
script_risk_level   = "LOW"
script_version      = "1.0.0"
script_type         = "SHELL"
script_content      = <<EOF
#! /bin/bash
echo "hello world!"
EOF

# 脚本参数配置
script_parameters = [
  {
    name        = "name"
    value       = "world"
    description = "the parameter"
  }
]

# 脚本执行配置
script_execute_timeout      = 600
script_execute_execute_user = "root"
script_execute_parameters = [
  {
    name  = "name"
    value = "somebody"
  }
]
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

### 12. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建COC脚本执行
4. 运行 `terraform show` 查看已创建的COC脚本执行

## 参考信息

- [华为云COC产品文档](https://support.huaweicloud.com/coc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [COC脚本执行最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/coc/script-execution)
