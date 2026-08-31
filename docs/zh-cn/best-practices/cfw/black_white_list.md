# 部署黑白名单

## 应用场景

云防火墙（Cloud Firewall，CFW）是新一代的云原生防火墙，提供云上互联网边界和VPC边界的防护，包括实时入侵检测与防御、全局统一访问控制、全流量分析可视化、日志审计与溯源分析等能力。CFW支持通过黑白名单对指定IP地址、端口和协议进行访问控制，黑名单用于阻断恶意流量，白名单用于放行可信流量。

本最佳实践将介绍如何使用Terraform自动化部署CFW黑白名单规则，包括查询已有防火墙信息，以及创建黑名单和白名单规则以分别阻断和放行指定IP地址。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [CFW防火墙列表查询数据源（data.huaweicloud_cfw_firewalls）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cfw_firewalls)

### 资源

- [CFW黑白名单资源（huaweicloud_cfw_black_white_list）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_black_white_list)

### 资源/数据源依赖关系

```
data.huaweicloud_cfw_firewalls
    └── huaweicloud_cfw_black_white_list
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询防火墙信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建CFW黑白名单规则：

```hcl
variable "fw_instance_id" {
  description = "The firewall instance ID"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下防火墙信息，用于创建CFW黑白名单规则
data "huaweicloud_cfw_firewalls" "test" {
  fw_instance_id = var.fw_instance_id != "" ? var.fw_instance_id : null
}

locals {
  object_id = try(data.huaweicloud_cfw_firewalls.test.records[0].protect_objects[0].object_id, null)
}
```

**参数说明**：
- **fw_instance_id**：防火墙实例ID，通过引用输入变量fw_instance_id进行赋值，当值为空字符串时设置为null以查询默认防火墙

### 3. 创建黑白名单规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CFW黑白名单规则资源：

```hcl
variable "blacklist_list_type" {
  description = "The list type of the blacklist rule. 4: blacklist, 5: whitelist"
  type        = number
  default     = 4
}

variable "whitelist_list_type" {
  description = "The list type of the whitelist rule. 4: blacklist, 5: whitelist"
  type        = number
  default     = 5
}

variable "blacklist_direction" {
  description = "The direction of the blacklist rule. 0: inbound, 1: outbound"
  type        = number
  default     = 0
}

variable "whitelist_direction" {
  description = "The direction of the whitelist rule. 0: inbound, 1: outbound"
  type        = number
  default     = 0
}

variable "blacklist_protocol" {
  description = "The protocol type of the blacklist rule. 6: TCP, 17: UDP, -1: any"
  type        = number
  default     = 6
}

variable "whitelist_protocol" {
  description = "The protocol type of the whitelist rule. 6: TCP, 17: UDP, -1: any"
  type        = number
  default     = 6
}

variable "blacklist_port" {
  description = "The destination port of the blacklist rule"
  type        = string
  default     = "22"
}

variable "whitelist_port" {
  description = "The destination port of the whitelist rule"
  type        = string
  default     = "80"
}

variable "blacklist_address_type" {
  description = "The IP address type of the blacklist rule. 0: IPv4, 1: IPv6"
  type        = number
  default     = 0
}

variable "whitelist_address_type" {
  description = "The IP address type of the whitelist rule. 0: IPv4, 1: IPv6"
  type        = number
  default     = 0
}

variable "blacklist_address" {
  description = "The IP address of the blacklist rule"
  type        = string
}

variable "whitelist_address" {
  description = "The IP address of the whitelist rule"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CFW黑白名单规则资源
resource "huaweicloud_cfw_black_white_list" "test" {
  count = 2

  object_id    = local.object_id
  list_type    = count.index == 0 ? var.blacklist_list_type : var.whitelist_list_type
  direction    = count.index == 0 ? var.blacklist_direction : var.whitelist_direction
  protocol     = count.index == 0 ? var.blacklist_protocol : var.whitelist_protocol
  port         = count.index == 0 ? var.blacklist_port : var.whitelist_port
  address_type = count.index == 0 ? var.blacklist_address_type : var.whitelist_address_type
  address      = count.index == 0 ? var.blacklist_address : var.whitelist_address
}
```

**参数说明**：
- **count**：黑白名单资源的创建数，设置为2表示分别创建一条黑名单规则和一条白名单规则
- **object_id**：防护对象ID，根据防火墙列表查询数据源（data.huaweicloud_cfw_firewalls）返回结果中的protect_objects进行赋值
- **list_type**：名单类型，当count.index为0时使用黑名单类型变量blacklist_list_type（默认为4），否则使用白名单类型变量whitelist_list_type（默认为5）
- **direction**：规则方向，当count.index为0时使用黑名单方向变量blacklist_direction（默认为0，入方向），否则使用白名单方向变量whitelist_direction（默认为0）
- **protocol**：协议类型，当count.index为0时使用黑名单协议变量blacklist_protocol（默认为6，TCP），否则使用白名单协议变量whitelist_protocol（默认为6）
- **port**：目的端口，当count.index为0时使用黑名单端口变量blacklist_port（默认为"22"），否则使用白名单端口变量whitelist_port（默认为"80"）
- **address_type**：IP地址类型，当count.index为0时使用黑名单地址类型变量blacklist_address_type（默认为0，IPv4），否则使用白名单地址类型变量whitelist_address_type（默认为0）
- **address**：IP地址，当count.index为0时通过引用输入变量blacklist_address进行赋值，否则通过引用输入变量whitelist_address进行赋值

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 防火墙实例
fw_instance_id = "your_firewall_instance_id"

# 黑名单规则配置
blacklist_list_type    = 4
blacklist_direction    = 0
blacklist_protocol     = 6
blacklist_port         = "22"
blacklist_address_type = 0
blacklist_address      = "1.1.1.1"

# 白名单规则配置
whitelist_list_type    = 5
whitelist_direction    = 0
whitelist_protocol     = 6
whitelist_port         = "443"
whitelist_address_type = 0
whitelist_address      = "2.2.2.2"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="fw_instance_id=your-firewall-id" -var="blacklist_address=1.1.1.1"`
2. 环境变量：`export TF_VAR_blacklist_address=1.1.1.1`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CFW黑白名单规则
4. 运行 `terraform show` 查看已创建的CFW黑白名单规则

## 参考信息

- [华为云CFW产品文档](https://support.huaweicloud.com/cfw/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CFW黑白名单最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cfw/black-white-list)
