# 部署SFS Turbo类型存储库

## 应用场景

云备份（Cloud Backup and Recovery, CBR）是华为云提供的数据保护服务，为云上资源和云下资源提供简单易用的备份服务。当发生病毒入侵、人为误删除、软硬件故障等事件时，可将数据恢复到任意备份点。SFS Turbo类型存储库是CBR服务中的一种存储库类型，专门用于备份弹性文件服务（SFS Turbo）文件系统。

SFS Turbo类型存储库支持对SFS Turbo文件系统进行完整备份，确保在发生故障时能够快速恢复整个文件系统环境。SFS Turbo是华为云提供的高性能文件存储服务，专为高性能计算和AI/ML工作负载设计，通过CBR备份服务可以确保重要文件数据的安全性和可恢复性。本最佳实践将介绍如何使用Terraform自动化部署一个CBR SFS Turbo类型存储库，包括创建SFS Turbo文件系统、配置备份策略和创建存储库。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [SFS Turbo文件系统资源（huaweicloud_sfs_turbo）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sfs_turbo)
- [CBR备份策略资源（huaweicloud_cbr_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cbr_policy)
- [CBR存储库资源（huaweicloud_cbr_vault）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cbr_vault)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_sfs_turbo.test
            └── huaweicloud_cbr_vault.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_sfs_turbo.test

huaweicloud_cbr_policy.test
    └── huaweicloud_cbr_vault.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询SFS Turbo文件系统资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建SFS Turbo文件系统：

```hcl
variable "availability_zone" {
  description = "SFS Turbo文件系统所属的可用区信息"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建SFS Turbo文件系统
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 无特殊参数，获取当前region下所有可用区信息

### 3. 创建VPC

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

### 4. 创建VPC子网

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
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**参数说明**：
- **vpc_id**：VPC ID，通过引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网CIDR块，优先使用输入变量，如果为空则通过cidrsubnet函数计算
- **gateway_ip**：网关IP，优先使用输入变量，如果为空则通过cidrhost函数计算
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区列表查询数据源的第一个结果

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "secgroup_name" {
  description = "安全组名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.secgroup_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量secgroup_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true以删除默认的安全组规则

### 6. 创建SFS Turbo文件系统

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建SFS Turbo文件系统资源：

```hcl
variable "turbo_name" {
  description = "SFS Turbo文件系统名称"
  type        = string
}

variable "turbo_size" {
  description = "SFS Turbo文件系统大小（GB）"
  type        = number
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SFS Turbo文件系统资源
resource "huaweicloud_sfs_turbo" "test" {
  name              = var.turbo_name
  size              = var.turbo_size
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**参数说明**：
- **name**：SFS Turbo文件系统名称，通过引用输入变量turbo_name进行赋值
- **size**：文件系统大小，通过引用输入变量turbo_size进行赋值
- **vpc_id**：VPC ID，通过引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **subnet_id**：子网ID，通过引用VPC子网资源（huaweicloud_vpc_subnet.test）的ID进行赋值
- **security_group_id**：安全组ID，通过引用安全组资源（huaweicloud_networking_secgroup.test）的ID进行赋值
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区列表查询数据源的第一个结果

### 7. 创建CBR备份策略（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CBR备份策略资源：

```hcl
variable "enable_policy" {
  description = "是否启用备份策略"
  type        = bool
  default     = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CBR备份策略资源
resource "huaweicloud_cbr_policy" "test" {
  count = var.enable_policy ? 1 : 0

  name        = "${var.vault_name}-policy"
  type        = "backup"
  time_period = 20
  time_zone   = "UTC+08:00"
  enabled     = true

  backup_cycle {
    days            = "MO,TU"
    execution_times = ["06:00"]
  }
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建备份策略资源，仅当 `var.enable_policy` 为true时创建
- **name**：备份策略名称，通过引用输入变量vault_name和固定后缀进行赋值
- **type**：策略类型，设置为"backup"表示备份策略
- **time_period**：备份保留时间（天），设置为20天
- **time_zone**：时区，设置为"UTC+08:00"
- **enabled**：是否启用策略，设置为true
- **backup_cycle.days**：备份周期，设置为周一和周二
- **backup_cycle.execution_times**：执行时间，设置为06:00

### 8. 创建CBR存储库

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CBR存储库资源：

```hcl
variable "vault_name" {
  description = "CBR存储库名称"
  type        = string
}

variable "protection_type" {
  description = "存储库保护类型（backup或replication）"
  type        = string
  default     = "backup"

  validation {
    condition     = contains(["backup", "replication"], var.protection_type)
    error_message = "保护类型必须是'backup'或'replication'。"
  }
}

variable "vault_size" {
  description = "CBR存储库大小（GB）"
  type        = number
}

variable "auto_expand" {
  description = "存储库满时是否自动扩容"
  type        = bool
  default     = false
}

variable "enterprise_project_id" {
  description = "存储库所属企业项目ID"
  type        = string
  default     = "0"
}

variable "backup_name_prefix" {
  description = "备份名称前缀"
  type        = string
  default     = ""
}

variable "is_multi_az" {
  description = "存储库是否跨AZ部署"
  type        = bool
  default     = false
}

variable "tags" {
  description = "存储库标签"
  type        = map(string)
  default = {
    environment = "test"
    terraform   = "true"
    service     = "sfs-turbo"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CBR存储库资源
resource "huaweicloud_cbr_vault" "test" {
  name                  = var.vault_name
  type                  = "turbo"
  protection_type       = var.protection_type
  size                  = var.vault_size
  auto_expand           = var.auto_expand
  enterprise_project_id = var.enterprise_project_id
  backup_name_prefix    = var.backup_name_prefix
  is_multi_az           = var.is_multi_az

  resources {
    includes = [
      huaweicloud_sfs_turbo.test.id
    ]
  }

  dynamic "policy" {
    for_each = var.enable_policy ? [1] : []
    content {
      id = huaweicloud_cbr_policy.test[0].id
    }
  }

  tags = var.tags
}
```

**参数说明**：
- **name**：存储库名称，通过引用输入变量vault_name进行赋值
- **type**：存储库类型，设置为"turbo"表示SFS Turbo类型存储库
- **protection_type**：保护类型，通过引用输入变量protection_type进行赋值
- **size**：存储库大小，通过引用输入变量vault_size进行赋值
- **auto_expand**：是否自动扩容，通过引用输入变量auto_expand进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值
- **backup_name_prefix**：备份名称前缀，通过引用输入变量backup_name_prefix进行赋值
- **is_multi_az**：是否跨AZ部署，通过引用输入变量is_multi_az进行赋值
- **resources.includes**：包含的资源ID列表，通过引用SFS Turbo文件系统资源（huaweicloud_sfs_turbo.test）的ID进行赋值
- **policy.id**：策略ID，当启用策略时通过引用备份策略资源（huaweicloud_cbr_policy.test）的ID进行赋值
- **tags**：标签，通过引用输入变量tags进行赋值

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 网络配置
vpc_name    = "cbr-test-vpc"
subnet_name = "cbr-test-subnet"
secgroup_name = "cbr-test-sg"

# SFS Turbo文件系统配置
turbo_name = "cbr-test-turbo"
turbo_size = 500

# CBR存储库配置
vault_name        = "cbr-vault-turbo"
vault_size        = 1000
enable_policy     = true
protection_type   = "backup"
auto_expand       = false
is_multi_az       = true

# 标签配置
tags = {
  environment = "test"
  project     = "cbr-sfs-turbo-demo"
  terraform   = "true"
  service     = "sfs-turbo"
}
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

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CBR SFS Turbo类型存储库
4. 运行 `terraform show` 查看已创建的CBR SFS Turbo类型存储库

## 参考信息

- [华为云CBR产品文档](https://support.huaweicloud.com/cbr/index.html)
- [华为云SFS Turbo产品文档](https://support.huaweicloud.com/sfsturbo/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CBR最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cbr)
