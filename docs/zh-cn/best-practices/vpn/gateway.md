# 部署VPN网关

## 应用场景

虚拟专用网络（Virtual Private Network，VPN）是华为云提供的安全、可靠的网络连接服务，支持在VPC与本地网络之间建立加密的IPsec VPN连接，实现云上资源与本地数据中心的互联互通。VPN网关是VPN服务的核心组件，提供高可用的VPN连接能力，支持双EIP部署，确保VPN连接的稳定性和可靠性。

本最佳实践将介绍如何使用Terraform自动化部署一个VPN网关，包括VPN网关可用区查询、VPC、子网、EIP和VPN网关的创建。通过VPN网关，您可以实现云上VPC与本地网络之间的安全连接，构建混合云架构。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [VPN网关可用区查询数据源（data.huaweicloud_vpn_gateway_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/vpn_gateway_availability_zones)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [EIP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [VPN网关资源（huaweicloud_vpn_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpn_gateway)

### 资源/数据源依赖关系

```
data.huaweicloud_vpn_gateway_availability_zones
    └── huaweicloud_vpn_gateway

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_vpn_gateway

huaweicloud_vpc_eip
    └── huaweicloud_vpn_gateway
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询VPN网关可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建VPN网关资源：

```hcl
variable "vpn_gateway_flavor" {
  description = "VPN网关的规格"
  type        = string
  default     = "professional1"
}

variable "vpn_gateway_attachment_type" {
  description = "VPC的关联类型"
  type        = string
  default     = "vpc"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合条件的VPN网关可用区信息，用于创建VPN网关资源
data "huaweicloud_vpn_gateway_availability_zones" "test" {
  flavor          = var.vpn_gateway_flavor
  attachment_type = var.vpn_gateway_attachment_type
}
```

**参数说明**：
- **flavor**：VPN网关的规格，通过引用输入变量 `vpn_gateway_flavor` 进行赋值，默认为"professional1"表示专业版1型
- **attachment_type**：VPC的关联类型，通过引用输入变量 `vpn_gateway_attachment_type` 进行赋值，默认为"vpc"表示VPC类型

### 3. 创建VPC资源

在TF文件中添加以下脚本以告知Terraform创建VPC资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署VPN网关
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC的名称，通过引用输入变量 `vpc_name` 进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量 `vpc_cidr` 进行赋值，默认为"192.168.0.0/16"网段

### 4. 创建VPC子网资源

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署VPN网关
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量 `subnet_name` 进行赋值
- **cidr**：子网的CIDR网段，如果指定了子网CIDR则使用该值，否则使用cidrsubnet函数从VPC的CIDR网段中划分一个子网段
- **gateway_ip**：子网的网关IP，如果指定了网关IP则使用该值，否则使用cidrhost函数从子网网段中获取第一个IP地址作为网关IP

### 5. 创建EIP资源

在TF文件中添加以下脚本以告知Terraform创建EIP资源（VPN网关需要两个EIP以实现高可用）：

```hcl
variable "eip_type" {
  description = "EIP的类型"
  type        = string
  default     = "5_bgp"
}

variable "bandwidth_name" {
  description = "带宽的名称"
  type        = string
}

variable "bandwidth_size" {
  description = "带宽的大小"
  type        = number
  default     = 8
}

variable "bandwidth_share_type" {
  description = "带宽的共享类型"
  type        = string
  default     = "PER"
}

variable "bandwidth_charge_mode" {
  description = "带宽的计费模式"
  type        = string
  default     = "traffic"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建EIP资源，用于部署VPN网关
resource "huaweicloud_vpc_eip" "test" {
  count = 2

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = "${var.bandwidth_name}-${count.index}"
    size        = var.bandwidth_size
    share_type  = var.bandwidth_share_type
    charge_mode = var.bandwidth_charge_mode
  }
}
```

**参数说明**：
- **count**：资源的创建数，设置为2表示创建两个EIP资源，用于VPN网关的高可用部署
- **publicip**：公网IP配置块
  - **type**：EIP类型，通过引用输入变量 `eip_type` 进行赋值，默认为"5_bgp"表示全动态BGP
- **bandwidth**：带宽配置块
  - **name**：带宽的名称，通过引用输入变量 `bandwidth_name` 和count索引进行赋值，确保每个EIP的带宽名称唯一
  - **size**：带宽的大小（Mbps），通过引用输入变量 `bandwidth_size` 进行赋值，默认为8Mbps
  - **share_type**：带宽的共享类型，通过引用输入变量 `bandwidth_share_type` 进行赋值，默认为"PER"表示独享
  - **charge_mode**：带宽的计费模式，通过引用输入变量 `bandwidth_charge_mode` 进行赋值，默认为"traffic"表示按流量计费

> 注意：VPN网关需要两个EIP以实现高可用部署，确保VPN连接的稳定性和可靠性。

### 6. 创建VPN网关资源

在TF文件中添加以下脚本以告知Terraform创建VPN网关资源：

```hcl
variable "vpn_gateway_name" {
  description = "VPN网关的名称"
  type        = string
}

variable "vpn_gateway_delete_eip_on_termination" {
  description = "删除VPN网关时是否删除EIP"
  type        = bool
  default     = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPN网关资源
resource "huaweicloud_vpn_gateway" "test" {
  name               = var.vpn_gateway_name
  vpc_id             = huaweicloud_vpc.test.id
  local_subnets      = [huaweicloud_vpc_subnet.test.cidr]
  connect_subnet     = huaweicloud_vpc_subnet.test.id
  availability_zones = [
    try(data.huaweicloud_vpn_gateway_availability_zones.test.names[0], "default_value"),
    try(data.huaweicloud_vpn_gateway_availability_zones.test.names[1], "default_value")
  ]

  eip1 {
    id = huaweicloud_vpc_eip.test[0].id
  }

  eip2 {
    id = huaweicloud_vpc_eip.test[1].id
  }

  delete_eip_on_termination = var.vpn_gateway_delete_eip_on_termination
}
```

**参数说明**：
- **name**：VPN网关的名称，通过引用输入变量 `vpn_gateway_name` 进行赋值
- **vpc_id**：VPN网关所属的VPC的ID，引用前面创建的VPC资源的ID
- **local_subnets**：VPN网关的本地子网列表，引用子网资源的CIDR网段
- **connect_subnet**：VPN网关的连接子网ID，引用子网资源的ID
- **availability_zones**：VPN网关所在的可用区列表，根据VPN网关可用区查询数据源的返回结果进行赋值，使用前两个可用区以实现高可用
- **eip1**：第一个EIP配置块
  - **id**：第一个EIP的ID，引用第一个EIP资源的ID
- **eip2**：第二个EIP配置块
  - **id**：第二个EIP的ID，引用第二个EIP资源的ID
- **delete_eip_on_termination**：删除VPN网关时是否删除EIP，通过引用输入变量 `vpn_gateway_delete_eip_on_termination` 进行赋值，默认为false表示不删除EIP

> 注意：VPN网关需要配置两个EIP以实现高可用部署，两个EIP分别部署在不同的可用区，确保VPN连接的稳定性和可靠性。

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 网络资源配置
vpc_name         = "tf_test_vpc"
subnet_name      = "tf_test_subnet"
bandwidth_name   = "tf_test_bandwidth"

# VPN网关配置
vpn_gateway_name = "tf_test_gateway"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="vpn_gateway_name=my-gateway"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建VPN网关
4. 运行 `terraform show` 查看已创建的VPN网关详情

## 参考信息

- [华为云VPN产品文档](https://support.huaweicloud.com/vpn/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VPN网关最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpn/gateway)
