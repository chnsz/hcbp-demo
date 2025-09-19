# 部署绑定EIP的MySQL单机实例

## 应用场景

华为云关系型数据库服务（RDS）的绑定EIP的MySQL单机实例功能提供具有公网访问能力的MySQL数据库服务，支持通过弹性公网IP（EIP）从互联网访问数据库实例。通过配置EIP绑定，您可以为RDS实例提供公网连接能力，满足跨地域访问、远程开发、数据同步等场景的需求。

本最佳实践特别适用于需要公网访问MySQL数据库、实现跨地域数据同步、支持远程开发和测试的场景，如多地域应用部署、远程办公、第三方系统集成等。本最佳实践将介绍如何使用Terraform自动化部署绑定EIP的RDS MySQL单机实例，包括VPC网络、安全组、RDS实例、EIP和EIP绑定的创建，实现完整的公网可访问MySQL数据库解决方案。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS规格查询数据源（data.huaweicloud_rds_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)
- [网络端口查询数据源（data.huaweicloud_networking_port）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/networking_port)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [随机密码资源（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [RDS实例资源（huaweicloud_rds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [弹性公网IP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [EIP绑定资源（huaweicloud_vpc_eip_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip_associate)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_rds_instance

data.huaweicloud_rds_flavors
    └── huaweicloud_rds_instance

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_rds_instance

huaweicloud_vpc_subnet
    ├── huaweicloud_rds_instance
    └── data.huaweicloud_networking_port

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_rds_instance

huaweicloud_networking_secgroup_rule
    └── huaweicloud_rds_instance

random_password
    └── huaweicloud_rds_instance

huaweicloud_rds_instance
    └── data.huaweicloud_networking_port

huaweicloud_vpc_eip
    └── huaweicloud_vpc_eip_associate

data.huaweicloud_networking_port
    └── huaweicloud_vpc_eip_associate
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 前置资源准备

本最佳实践需要先创建VPC、子网、安全组和RDS实例等前置资源。请按照[部署MySQL单机实例](mysql_single_instance.md)最佳实践中的以下步骤进行准备：

- **步骤2**：创建VPC资源
- **步骤3**：查询可用区信息
- **步骤4**：创建VPC子网
- **步骤5**：查询RDS规格信息
- **步骤6**：创建安全组
- **步骤7**：创建安全组规则
- **步骤8**：创建随机密码
- **步骤9**：创建RDS实例

完成上述步骤后，继续执行本最佳实践的后续步骤。

### 3. 创建弹性公网IP

在TF文件中添加以下脚本以告知Terraform创建弹性公网IP资源：

```hcl
variable "associate_eip_address" {
  description = "The EIP address to associate with the RDS instance"
  type        = string
  default     = ""
}

variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "bandwidth_name" {
  description = "The name for the bandwidth"
  type        = string
  default     = ""

  validation {
    condition     = var.associate_eip_address != "" || var.bandwidth_name != ""
    error_message = "The bandwidth name must be a non-empty string if the EIP address is not provided."
  }
}

variable "bandwidth_size" {
  description = "The size of the bandwidth"
  type        = number
  default     = 5
}

variable "bandwidth_share_type" {
  description = "The share type of the bandwidth"
  type        = string
  default     = "PER"
}

variable "bandwidth_charge_mode" {
  description = "The charge mode of the bandwidth"
  type        = string
  default     = "traffic"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源
resource "huaweicloud_vpc_eip" "test" {
  count = var.associate_eip_address == "" ? 1 : 0

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.bandwidth_name
    size        = var.bandwidth_size
    share_type  = var.bandwidth_share_type
    charge_mode = var.bandwidth_charge_mode
  }
}
```

**参数说明**：
- **count**：条件创建，当associate_eip_address变量为空字符串时创建此资源
- **publicip**：公网IP配置块
  - **type**：EIP类型，通过引用输入变量eip_type进行赋值，默认值为"5_bgp"
- **bandwidth**：带宽配置块
  - **name**：带宽名称，通过引用输入变量bandwidth_name进行赋值，默认值为空字符串
  - **size**：带宽大小，通过引用输入变量bandwidth_size进行赋值，默认值为5（Mbps）
  - **share_type**：带宽共享类型，通过引用输入变量bandwidth_share_type进行赋值，默认值为"PER"
  - **charge_mode**：带宽计费模式，通过引用输入变量bandwidth_charge_mode进行赋值，默认值为"traffic"

### 4. 查询网络端口信息

在TF文件中添加以下脚本以告知Terraform查询网络端口信息：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的网络端口信息，用于EIP绑定
data "huaweicloud_networking_port" "test" {
  network_id = huaweicloud_vpc_subnet.test.id
  fixed_ip   = huaweicloud_rds_instance.test.fixed_ip

  depends_on = [
    huaweicloud_rds_instance.test
  ]
}
```

**参数说明**：
- **network_id**：网络ID，引用前面创建的VPC子网资源的ID
- **fixed_ip**：固定IP地址，引用前面创建的RDS实例资源的固定IP
- **depends_on**：显式依赖关系，确保RDS实例在端口查询前已创建

### 5. 创建EIP绑定

在TF文件中添加以下脚本以告知Terraform创建EIP绑定资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建EIP绑定资源
resource "huaweicloud_vpc_eip_associate" "test" {
  public_ip = var.associate_eip_address != "" ? var.associate_eip_address : huaweicloud_vpc_eip.test[0].address
  port_id   = data.huaweicloud_networking_port.test.id
}
```

**参数说明**：
- **public_ip**：公网IP地址，优先使用associate_eip_address变量，如果为空则使用新创建的EIP地址
- **port_id**：端口ID，引用查询到的网络端口资源的ID

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC网络配置
vpc_name = "tf_test_vpc"

# 子网配置
subnet_name = "tf_test_subnet"

# 安全组配置
security_group_name = "tf_test_security_group"

# RDS实例配置
instance_name = "tf_test_mysql_instance"

# 备份配置
instance_backup_time_window = "08:00-09:00"
instance_backup_keep_days   = 1

# EIP配置
bandwidth_name = "tf_test_bandwidth"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="instance_name=my-instance"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建绑定EIP的MySQL单机实例
4. 运行 `terraform show` 查看已创建的绑定EIP的MySQL单机实例详情

## 参考信息

- [华为云关系型数据库服务产品文档](https://support.huaweicloud.com/rds/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RDS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rds)
