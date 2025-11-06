# 部署磁盘类型存储库

## 应用场景

云备份（Cloud Backup and Recovery, CBR）是华为云提供的数据保护服务，为云上资源和云下资源提供简单易用的备份服务。当发生病毒入侵、人为误删除、软硬件故障等事件时，可将数据恢复到任意备份点。磁盘类型存储库是CBR服务中的一种存储库类型，专门用于备份云硬盘（EVS）卷。

磁盘类型存储库支持对云硬盘卷进行完整备份，确保在发生故障时能够快速恢复整个磁盘数据。云硬盘是华为云提供的可扩展虚拟块存储服务，具有高可靠、高性能、易扩展等特点，通过CBR备份服务可以确保重要磁盘数据的安全性和可恢复性。本最佳实践将介绍如何使用Terraform自动化部署一个CBR磁盘类型存储库，包括创建云硬盘卷和创建存储库。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [云硬盘卷资源（huaweicloud_evs_volume）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)
- [CBR存储库资源（huaweicloud_cbr_vault）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cbr_vault)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_evs_volume.test
        └── huaweicloud_cbr_vault.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询云硬盘卷资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建云硬盘卷：

```hcl
variable "availability_zone" {
  description = "云硬盘卷所属的可用区信息"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建云硬盘卷
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 无特殊参数，获取当前region下所有可用区信息

### 3. 创建云硬盘卷

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云硬盘卷资源：

```hcl
variable "volume_type" {
  description = "云硬盘卷类型"
  type        = string
}

variable "volume_name" {
  description = "云硬盘卷名称"
  type        = string
  default     = ""
}

variable "volume_size" {
  description = "云硬盘卷大小（GB）"
  type        = number
}

variable "volume_device_type" {
  description = "云硬盘卷设备类型"
  type        = string
  default     = "VBD"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云硬盘卷资源
resource "huaweicloud_evs_volume" "test" {
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  volume_type       = var.volume_type
  name              = var.volume_name
  size              = var.volume_size
  device_type       = var.volume_device_type
}
```

**参数说明**：
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区列表查询数据源的第一个结果
- **volume_type**：云硬盘卷类型，通过引用输入变量volume_type进行赋值
- **name**：云硬盘卷名称，通过引用输入变量volume_name进行赋值
- **size**：云硬盘卷大小，通过引用输入变量volume_size进行赋值
- **device_type**：云硬盘卷设备类型，通过引用输入变量volume_device_type进行赋值

### 4. 创建CBR存储库

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CBR存储库资源：

```hcl
variable "name" {
  description = "CBR存储库名称"
  type        = string
}

variable "type" {
  description = "CBR存储库类型"
  type        = string
  default     = "disk"
}

variable "protection_type" {
  description = "存储库保护类型"
  type        = string
  default     = "backup"
}

variable "size" {
  description = "CBR存储库大小（GB）"
  type        = number
}

variable "enterprise_project_id" {
  description = "存储库所属企业项目ID"
  type        = string
  default     = "0"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CBR存储库资源
resource "huaweicloud_cbr_vault" "test" {
  name                  = var.name
  type                  = var.type
  protection_type       = var.protection_type
  size                  = var.size
  enterprise_project_id = var.enterprise_project_id

  resources {
    includes = [huaweicloud_evs_volume.test.id]
  }
}
```

**参数说明**：
- **name**：存储库名称，通过引用输入变量name进行赋值
- **type**：存储库类型，通过引用输入变量type进行赋值，默认为"disk"表示磁盘类型存储库
- **protection_type**：保护类型，通过引用输入变量protection_type进行赋值
- **size**：存储库大小，通过引用输入变量size进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值
- **resources.includes**：包含的资源ID列表，通过引用云硬盘卷资源（huaweicloud_evs_volume.test）的ID进行赋值

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 云硬盘卷配置
volume_type = "SSD"
volume_size = 50
volume_name = "cbr-test-volume"
volume_device_type = "VBD"

# CBR存储库配置
name        = "tf_cbr_script"
size        = 100
type        = "disk"
protection_type = "backup"
enterprise_project_id = "0"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="volume_type=SSD" -var="volume_size=50"`
2. 环境变量：`export TF_VAR_volume_type=SSD`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CBR磁盘类型存储库
4. 运行 `terraform show` 查看已创建的CBR磁盘类型存储库

## 参考信息

- [华为云CBR产品文档](https://support.huaweicloud.com/cbr/index.html)
- [华为云EVS产品文档](https://support.huaweicloud.com/evs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CBR磁盘类型存储库最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cbr/vault-volume)
