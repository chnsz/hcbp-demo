# 部署连接

## 应用场景

虚拟专用网络（Virtual Private Network，VPN）是华为云提供的安全、可靠的网络连接服务，支持在VPC与本地网络之间建立加密的IPsec VPN连接，实现云上资源与本地数据中心的互联互通。VPN连接是VPN服务的核心功能，用于在VPN网关和客户网关之间建立IPsec VPN连接，实现云上VPC与本地网络之间的安全通信。通过VPN连接，可以建立加密的隧道，确保数据传输的安全性和稳定性，支持多种VPN类型和配置选项，满足不同场景的网络连接需求。本最佳实践将介绍如何使用Terraform自动化部署VPN连接，包括VPN网关可用区查询、VPC、子网、EIP、VPN网关、客户网关和VPN连接的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [VPN网关可用区查询数据源（data.huaweicloud_vpn_gateway_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/vpn_gateway_availability_zones)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [EIP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [VPN网关资源（huaweicloud_vpn_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpn_gateway)
- [客户网关资源（huaweicloud_vpn_customer_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpn_customer_gateway)
- [VPN连接资源（huaweicloud_vpn_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpn_connection)

### 资源/数据源依赖关系

```text
data.huaweicloud_vpn_gateway_availability_zones
    └── huaweicloud_vpn_gateway
        └── huaweicloud_vpn_connection

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_vpn_gateway
            └── huaweicloud_vpn_connection

huaweicloud_vpc_eip
    └── huaweicloud_vpn_gateway
        └── huaweicloud_vpn_connection

huaweicloud_vpn_customer_gateway
    └── huaweicloud_vpn_connection
```

> 注意：VPN连接需要依赖VPN网关和客户网关。VPN网关需要依赖VPC、子网和EIP资源。VPN网关可用区查询用于获取VPN网关可用的可用区信息。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询VPN网关可用区

在TF文件（如main.tf）中添加以下脚本以查询VPN网关可用区：

```hcl
# 查询VPN网关可用区数据源
data "huaweicloud_vpn_gateway_availability_zones" "test" {
  flavor          = var.vpn_gateway_az_flavor
  attachment_type = var.vpn_gateway_az_attachment_type
}
```

**参数说明**：
- **flavor**：VPN网关规格名称，通过引用输入变量 `vpn_gateway_az_flavor` 进行赋值
- **attachment_type**：VPN网关连接类型，通过引用输入变量 `vpn_gateway_az_attachment_type` 进行赋值

### 3. 创建VPC和子网

在TF文件（如main.tf）中添加以下脚本以创建VPC和子网：

```hcl
# 创建VPC资源
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

# 创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 `vpc_name` 进行赋值
- **cidr**：VPC的CIDR地址块，通过引用输入变量 `vpc_cidr` 进行赋值
- **vpc_id**：子网所属的VPC ID，通过引用VPC资源的ID进行赋值
- **cidr**：子网的CIDR地址块，如果输入变量为空则自动计算，否则使用输入变量的值
- **gateway_ip**：子网的网关IP地址，如果输入变量为空则自动计算，否则使用输入变量的值

### 4. 创建EIP

在TF文件（如main.tf）中添加以下脚本以创建EIP：

```hcl
# 创建EIP资源（需要创建2个EIP用于VPN网关的双EIP部署）
resource "huaweicloud_vpc_eip" "test" {
  count = 2

  dynamic "publicip" {
    for_each = var.vpc_eip_public_ip

    content {
      type = publicip.value.type
    }
  }

  dynamic "bandwidth" {
    for_each = var.vpc_eip_bandwidth

    content {
      name        = bandwidth.value.name
      size        = bandwidth.value.size
      share_type  = bandwidth.value.share_type
      charge_mode = bandwidth.value.charge_mode
    }
  }
}
```

**参数说明**：
- **publicip**：公网IP配置，通过动态块 `dynamic "publicip"` 根据输入变量 `vpc_eip_public_ip` 创建公网IP配置
  - **type**：公网IP类型，通过引用输入变量中的 `type` 进行赋值
- **bandwidth**：带宽配置，通过动态块 `dynamic "bandwidth"` 根据输入变量 `vpc_eip_bandwidth` 创建带宽配置
  - **name**：带宽名称，通过引用输入变量中的 `name` 进行赋值
  - **size**：带宽大小，通过引用输入变量中的 `size` 进行赋值
  - **share_type**：带宽共享类型，通过引用输入变量中的 `share_type` 进行赋值
  - **charge_mode**：带宽计费模式，通过引用输入变量中的 `charge_mode` 进行赋值

> 注意：VPN网关需要2个EIP用于双EIP部署，确保VPN连接的高可用性。

### 5. 创建VPN网关

在TF文件（如main.tf）中添加以下脚本以创建VPN网关：

```hcl
# 创建VPN网关资源
resource "huaweicloud_vpn_gateway" "test" {
  name               = var.vpn_gateway_name
  vpc_id             = huaweicloud_vpc.test.id
  local_subnets      = [huaweicloud_vpc_subnet.test.cidr]
  connect_subnet     = huaweicloud_vpc_subnet.test.id
  availability_zones = [
    try(data.huaweicloud_vpn_gateway_availability_zones.test.names[0], null),
    try(data.huaweicloud_vpn_gateway_availability_zones.test.names[1], null)
  ]

  eip1 {
    id = huaweicloud_vpc_eip.test[0].id
  }

  eip2 {
    id = huaweicloud_vpc_eip.test[1].id
  }
}
```

**参数说明**：
- **name**：VPN网关名称，通过引用输入变量 `vpn_gateway_name` 进行赋值
- **vpc_id**：VPN网关所属的VPC ID，通过引用VPC资源的ID进行赋值
- **local_subnets**：VPN网关的本地子网列表，通过引用子网资源的CIDR进行赋值
- **connect_subnet**：VPN网关的连接子网ID，通过引用子网资源的ID进行赋值
- **availability_zones**：VPN网关的可用区列表，通过引用VPN网关可用区查询数据源的结果进行赋值
- **eip1**：VPN网关的第一个EIP配置，通过引用第一个EIP资源的ID进行赋值
- **eip2**：VPN网关的第二个EIP配置，通过引用第二个EIP资源的ID进行赋值

### 6. 创建客户网关

在TF文件（如main.tf）中添加以下脚本以创建客户网关：

```hcl
# 创建客户网关资源
resource "huaweicloud_vpn_customer_gateway" "test" {
  name     = var.vpn_customer_gateway_name
  id_value = var.vpn_customer_gateway_id_value
}
```

**参数说明**：
- **name**：客户网关名称，通过引用输入变量 `vpn_customer_gateway_name` 进行赋值
- **id_value**：客户网关的标识符（通常是公网IP地址），通过引用输入变量 `vpn_customer_gateway_id_value` 进行赋值

### 7. 创建VPN连接

在TF文件（如main.tf）中添加以下脚本以创建VPN连接：

```hcl
# 创建VPN连接资源
resource "huaweicloud_vpn_connection" "test" {
  name                = var.vpn_connection_name
  gateway_id          = huaweicloud_vpn_gateway.test.id
  gateway_ip          = huaweicloud_vpn_gateway.test.master_eip[0].id
  customer_gateway_id = huaweicloud_vpn_customer_gateway.test.id
  peer_subnets        = var.vpn_connection_peer_subnets
  vpn_type            = var.vpn_connection_vpn_type
  psk                 = var.vpn_connection_psk
  enable_nqa          = var.vpn_connection_enable_nqa
}
```

**参数说明**：
- **name**：VPN连接名称，通过引用输入变量 `vpn_connection_name` 进行赋值
- **gateway_id**：VPN网关ID，通过引用VPN网关资源的ID进行赋值
- **gateway_ip**：VPN网关IP地址，通过引用VPN网关资源的主EIP ID进行赋值
- **customer_gateway_id**：客户网关ID，通过引用客户网关资源的ID进行赋值
- **peer_subnets**：对端子网列表，通过引用输入变量 `vpn_connection_peer_subnets` 进行赋值
- **vpn_type**：VPN连接类型，通过引用输入变量 `vpn_connection_vpn_type` 进行赋值
- **psk**：预共享密钥，通过引用输入变量 `vpn_connection_psk` 进行赋值
- **enable_nqa**：是否启用NQA检测，通过引用输入变量 `vpn_connection_enable_nqa` 进行赋值

> 注意：VPN连接需要在VPN网关和客户网关创建完成后才能创建。预共享密钥（PSK）需要与对端网关配置保持一致，确保VPN连接能够正常建立。

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPN网关可用区查询配置（可选）
vpn_gateway_az_flavor          = "professional1"
vpn_gateway_az_attachment_type = "vpc"

# VPC和子网配置（必填）
vpc_name   = "tf_test_vpn_connection"
vpc_cidr   = "192.168.0.0/16"
subnet_name = "tf_test_vpn_connection"

# VPN网关配置（必填）
vpn_gateway_name = "tf_test_vpn_connection"

# 客户网关配置（必填）
vpn_customer_gateway_name  = "tf_test_vpn_connection"
vpn_customer_gateway_id_value = "8.8.8.8"

# VPN连接配置（必填）
vpn_connection_name         = "tf_test_vpn_connection"
vpn_connection_peer_subnets = ["10.0.0.0/24"]
vpn_connection_vpn_type     = "ipsec"
vpn_connection_psk           = "test_psk_123456"
vpn_connection_enable_nqa    = true
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=tf_test_vpn_connection"`
2. 环境变量：`export TF_VAR_vpc_name=tf_test_vpn_connection`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建VPN连接及相关资源
4. 运行 `terraform show` 查看已创建的VPN连接

## 参考信息

- [华为云VPN产品文档](https://support.huaweicloud.com/vpn/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VPN连接最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpn/connection)
