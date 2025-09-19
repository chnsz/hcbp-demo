# 部署文件系统

## 应用场景

华为云弹性文件服务（SFS Turbo）的文件系统功能提供高性能、高可用的文件存储服务，专为高性能计算（HPC）、AI/ML工作负载和大规模数据处理场景设计。SFS Turbo支持NFS和CIFS协议，提供高IOPS、低延迟的文件访问能力，能够满足企业级应用的存储需求。

本最佳实践特别适用于需要高性能文件存储、支持大规模并发访问、实现数据共享和协作的场景，如HPC集群、机器学习训练、大数据分析、容器化应用存储等。本最佳实践将介绍如何使用Terraform自动化部署SFS Turbo文件系统，包括VPC网络、安全组、SFS Turbo文件系统的创建，实现完整的文件存储解决方案。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [SFS Turbo文件系统资源（huaweicloud_sfs_turbo）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sfs_turbo)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_sfs_turbo

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_sfs_turbo

huaweicloud_vpc_subnet
    └── huaweicloud_sfs_turbo

huaweicloud_networking_secgroup
    └── huaweicloud_sfs_turbo
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询可用区信息：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有可用的可用区信息，用于创建SFS Turbo文件系统
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 此数据源无需额外参数，会自动查询当前区域的所有可用区

### 3. 创建VPC网络

在TF文件中添加以下脚本以告知Terraform创建VPC资源：

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
- **name**：VPC的名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"

### 4. 创建VPC子网

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR块，如果subnet_cidr为空则自动计算，否则使用subnet_cidr的值
- **gateway_ip**：子网的网关IP，如果subnet_gateway_ip为空则自动计算，否则使用subnet_gateway_ip的值

### 5. 创建安全组

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

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
- **name**：安全组的名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认安全组规则

### 6. 创建SFS Turbo文件系统

在TF文件中添加以下脚本以告知Terraform创建SFS Turbo文件系统资源：

```hcl
variable "turbo_name" {
  description = "The name of the SFS Turbo file system"
  type        = string
  default     = ""
}

variable "turbo_size" {
  description = "The capacity of the SFS Turbo file system"
  type        = number
  default     = 1228
}

variable "turbo_share_proto" {
  description = "The protocol of the SFS Turbo file system"
  type        = string
  default     = "NFS"
}

variable "turbo_share_type" {
  description = "The type of the SFS Turbo file system"
  type        = string
  default     = "STANDARD"
}

variable "turbo_hpc_bandwidth" {
  description = "The bandwidth specification of the SFS Turbo file system"
  type        = string
  default     = ""
}

variable "turbo_tags" {
  description = "The tags of the SFS Turbo file system"
  type        = map(string)
  default     = {}
}

variable "turbo_backup_id" {
  description = "The ID of the backup"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the SFS Turbo file system"
  type        = string
  default     = null
}

variable "charging_mode" {
  description = "The charging mode of the SFS Turbo file system"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the SFS Turbo file system"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the SFS Turbo file system"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the SFS Turbo file system"
  type        = string
  default     = "false"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SFS Turbo文件系统资源
resource "huaweicloud_sfs_turbo" "test" {
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zone     = try(data.huaweicloud_availability_zones.test.names[0], null)
  name                  = var.turbo_name
  size                  = var.turbo_size
  share_proto           = var.turbo_share_proto
  share_type            = var.turbo_share_type
  hpc_bandwidth         = var.turbo_hpc_bandwidth
  tags                  = var.turbo_tags
  backup_id             = var.turbo_backup_id
  enterprise_project_id = var.enterprise_project_id
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew

  # If you want to change some of the following parameters, you need to remove the corresponding fields from "lifecycle.ignore_changes".
  lifecycle {
    ignore_changes = [
      availability_zone,
    ]
  }
}
```

**参数说明**：
- **vpc_id**：VPC ID，引用前面创建的VPC资源的ID
- **subnet_id**：子网ID，引用前面创建的VPC子网资源的ID
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **availability_zone**：可用区，使用查询到的第一个可用区
- **name**：SFS Turbo文件系统的名称，通过引用输入变量turbo_name进行赋值，默认值为空字符串
- **size**：文件系统容量，通过引用输入变量turbo_size进行赋值，默认值为1228（GB）
- **share_proto**：共享协议，通过引用输入变量turbo_share_proto进行赋值，默认值为"NFS"
- **share_type**：共享类型，通过引用输入变量turbo_share_type进行赋值，默认值为"STANDARD"
- **hpc_bandwidth**：HPC带宽规格，通过引用输入变量turbo_hpc_bandwidth进行赋值，默认值为空字符串
- **tags**：文件系统标签，通过引用输入变量turbo_tags进行赋值，默认值为空映射
- **backup_id**：备份ID，通过引用输入变量turbo_backup_id进行赋值，默认值为空字符串
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null
- **charging_mode**：计费模式，通过引用输入变量charging_mode进行赋值，默认值为"postPaid"
- **period_unit**：计费周期单位，通过引用输入变量period_unit进行赋值，默认值为null
- **period**：计费周期，通过引用输入变量period进行赋值，默认值为null
- **auto_renew**：是否自动续费，通过引用输入变量auto_renew进行赋值，默认值为"false"
- **lifecycle.ignore_changes**：生命周期管理，忽略availability_zone的变更

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC网络配置
vpc_name = "tf_test_turbo"

# 子网配置
subnet_name = "tf_test_turbo"

# 安全组配置
security_group_name = "tf_test_turbo"

# SFS Turbo文件系统配置
turbo_name          = "tf_test_turbo_demo"
turbo_share_type    = "HPC"
turbo_hpc_bandwidth = "40M"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="turbo_name=my-turbo"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建文件系统
4. 运行 `terraform show` 查看已创建的文件系统详情

## 参考信息

- [华为云弹性文件服务产品文档](https://support.huaweicloud.com/sfs-turbo/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SFS Turbo最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sfs-turbo)
