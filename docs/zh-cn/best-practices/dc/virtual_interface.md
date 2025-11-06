# 部署虚拟接口

## 应用场景

云专线（Direct Connect, DC）是华为云提供的高性能、低延迟、安全可靠的专线接入服务，为企业提供从本地数据中心到华为云的专用网络连接。云专线服务支持多种接入方式，包括物理专线、虚拟专线等，满足不同规模和场景的网络连接需求。

虚拟接口是云专线服务中的核心组件，用于建立专线与VPC之间的逻辑连接。通过虚拟接口，可以实现专线流量的路由和转发，支持静态路由和BGP动态路由，满足不同网络架构的需求。本最佳实践将介绍如何使用Terraform自动化部署DC虚拟接口，包括VPC创建、虚拟网关创建和虚拟接口配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

无

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [DC虚拟网关资源（huaweicloud_dc_virtual_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dc_virtual_gateway)
- [DC虚拟接口资源（huaweicloud_dc_virtual_interface）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dc_virtual_interface)

### 资源/数据源依赖关系

```
huaweicloud_vpc.test
    └── huaweicloud_dc_virtual_gateway.test
        └── huaweicloud_dc_virtual_interface.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC

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

### 3. 创建DC虚拟网关

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DC虚拟网关资源：

```hcl
variable "virtual_gateway_name" {
  description = "虚拟网关名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DC虚拟网关资源
resource "huaweicloud_dc_virtual_gateway" "test" {
  vpc_id = huaweicloud_vpc.test.id
  name   = var.virtual_gateway_name

  local_ep_group = [
    huaweicloud_vpc.test.cidr,
  ]
}
```

**参数说明**：
- **vpc_id**：VPC ID，通过引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **name**：虚拟网关名称，通过引用输入变量virtual_gateway_name进行赋值
- **local_ep_group**：本地端点组，通过引用VPC资源（huaweicloud_vpc.test）的CIDR进行赋值

### 4. 创建DC虚拟接口

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DC虚拟接口资源：

```hcl
variable "direct_connect_id" {
  description = "与虚拟接口关联的专线连接ID"
  type        = string
}

variable "virtual_interface_name" {
  description = "虚拟接口名称"
  type        = string
}

variable "virtual_interface_description" {
  description = "虚拟接口描述"
  type        = string
  default     = "Created by Terraform"
}

variable "virtual_interface_type" {
  description = "虚拟接口类型"
  type        = string
  default     = "private"
}

variable "route_mode" {
  description = "虚拟接口路由模式"
  type        = string
  default     = "static"
}

variable "vlan" {
  description = "客户侧VLAN"
  type        = number
}

variable "bandwidth" {
  description = "虚拟接口入方向带宽大小"
  type        = number
}

variable "remote_ep_group" {
  description = "远程子网CIDR列表"
  type        = list(string)
}

variable "address_family" {
  description = "虚拟接口地址族类型"
  type        = string
  default     = "ipv4"
}

variable "local_gateway_v4_ip" {
  description = "云侧虚拟接口IPv4地址"
  type        = string
}

variable "remote_gateway_v4_ip" {
  description = "客户侧虚拟接口IPv4地址"
  type        = string
}

variable "enable_bfd" {
  description = "是否启用双向转发检测（BFD）功能"
  type        = bool
  default     = false
}

variable "enable_nqa" {
  description = "是否启用网络质量分析（NQA）功能"
  type        = bool
  default     = false
}

variable "virtual_interface_tags" {
  description = "虚拟接口标签"
  type        = map(string)
  default = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DC虚拟接口资源
resource "huaweicloud_dc_virtual_interface" "test" {
  direct_connect_id = var.direct_connect_id
  vgw_id            = huaweicloud_dc_virtual_gateway.test.id
  name              = var.virtual_interface_name
  description       = var.virtual_interface_description
  type              = var.virtual_interface_type
  route_mode        = var.route_mode
  vlan              = var.vlan
  bandwidth         = var.bandwidth

  remote_ep_group = var.remote_ep_group

  address_family       = var.address_family
  local_gateway_v4_ip  = var.local_gateway_v4_ip
  remote_gateway_v4_ip = var.remote_gateway_v4_ip

  enable_bfd = var.enable_bfd
  enable_nqa = var.enable_nqa

  tags = var.virtual_interface_tags
}
```

**参数说明**：
- **direct_connect_id**：专线连接ID，通过引用输入变量direct_connect_id进行赋值
- **vgw_id**：虚拟网关ID，通过引用DC虚拟网关资源（huaweicloud_dc_virtual_gateway.test）的ID进行赋值
- **name**：虚拟接口名称，通过引用输入变量virtual_interface_name进行赋值
- **description**：虚拟接口描述，通过引用输入变量virtual_interface_description进行赋值
- **type**：虚拟接口类型，通过引用输入变量virtual_interface_type进行赋值，支持"private"和"public"
- **route_mode**：路由模式，通过引用输入变量route_mode进行赋值，支持"static"和"bgp"
- **vlan**：客户侧VLAN，通过引用输入变量vlan进行赋值
- **bandwidth**：入方向带宽大小，通过引用输入变量bandwidth进行赋值
- **remote_ep_group**：远程子网CIDR列表，通过引用输入变量remote_ep_group进行赋值
- **address_family**：地址族类型，通过引用输入变量address_family进行赋值，支持"ipv4"和"ipv6"
- **local_gateway_v4_ip**：云侧IPv4地址，通过引用输入变量local_gateway_v4_ip进行赋值
- **remote_gateway_v4_ip**：客户侧IPv4地址，通过引用输入变量remote_gateway_v4_ip进行赋值
- **enable_bfd**：是否启用BFD功能，通过引用输入变量enable_bfd进行赋值
- **enable_nqa**：是否启用NQA功能，通过引用输入变量enable_nqa进行赋值
- **tags**：标签，通过引用输入变量virtual_interface_tags进行赋值

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC配置
vpc_name = "tf_test_vpc"
vpc_cidr = "192.168.0.0/16"

# 虚拟网关配置
virtual_gateway_name = "tf_test_virtual_gateway"

# 虚拟接口配置
virtual_interface_name        = "tf_test_virtual_interface"
virtual_interface_description = "Created by Terraform"
virtual_interface_type        = "private"
route_mode                    = "static"
direct_connect_id             = "f50a0a20-7214-4614-b2f8-830994186934"
vlan                          = 100
bandwidth                     = 100
remote_ep_group               = ["10.10.10.0/30"]
address_family                = "ipv4"
local_gateway_v4_ip           = "10.10.10.1/30"
remote_gateway_v4_ip          = "10.10.10.2/30"
enable_bfd                    = false
enable_nqa                    = false

# 标签配置
virtual_interface_tags = {
  "Owner"      = "terraform"
  "Env"        = "test"
  "Project"    = "dc-demo"
  "CostCenter" = "IT"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="virtual_interface_name=my-interface"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DC虚拟接口
4. 运行 `terraform show` 查看已创建的DC虚拟接口

## 参考信息

- [华为云DC产品文档](https://support.huaweicloud.com/dc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DC虚拟接口最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dc/virtual-interface)
