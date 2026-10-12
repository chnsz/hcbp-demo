# 部署VPC互联网网关

## 应用场景

VPC互联网网关（Internet Gateway）是华为云虚拟私有云中用于打通VPC与公网的关键组件，为VPC内的云资源提供访问公网以及被公网访问的能力。在跨区域IPv4网络（GEIP）等场景中，互联网网关是VPC实现公网互通的前置条件，只有先创建互联网网关，VPC内的资源才能通过公网进行通信。

本最佳实践将介绍如何使用Terraform自动化部署VPC互联网网关，包括VPC创建、子网创建以及互联网网关的创建与路由配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [VPC互联网网关（huaweicloud_vpc_internet_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_internet_gateway)

### 资源/数据源依赖关系

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_vpc_internet_gateway
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云资源
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，通过引用输入变量 vpc_cidr 进行赋值，默认值为`192.168.0.0/16`

### 3. 创建VPC子网

在TF文件（如main.tf）中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
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
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，通过引用资源 huaweicloud_vpc.test 的ID进行赋值
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网的网段，通过引用输入变量 subnet_cidr 进行赋值；当该变量为空字符串时，基于VPC网段自动划分
- **gateway_ip**：子网的网关IP地址，通过引用输入变量 subnet_gateway_ip 进行赋值；当该变量为空字符串时，基于子网网段自动计算

### 4. 创建VPC互联网网关

在TF文件（如main.tf）中添加以下脚本以创建互联网网关：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC互联网网关资源
variable "internet_gateway_name" {
  description = "The name of the VPC internet gateway"
  type        = string
}

variable "internet_gateway_add_route" {
  description = "Whether to add a default route pointing to the internet gateway"
  type        = bool
  default     = true
}

variable "internet_gateway_ipv6_enabled" {
  description = "Whether to enable IPv6 for the internet gateway"
  type        = bool
  default     = false
}

resource "huaweicloud_vpc_internet_gateway" "test" {
  vpc_id      = huaweicloud_vpc.test.id
  subnet_id   = huaweicloud_vpc_subnet.test.id
  name        = var.internet_gateway_name
  add_route   = var.internet_gateway_add_route
  enable_ipv6 = var.internet_gateway_ipv6_enabled
}
```

**参数说明**：
- **vpc_id**：互联网网关所属的VPC ID，通过引用资源 huaweicloud_vpc.test 的ID进行赋值
- **subnet_id**：互联网网关关联的子网ID，通过引用资源 huaweicloud_vpc_subnet.test 的ID进行赋值
- **name**：互联网网关名称，通过引用输入变量 internet_gateway_name 进行赋值
- **add_route**：是否添加指向互联网网关的默认路由，通过引用输入变量 internet_gateway_add_route 进行赋值，默认值为`true`
- **enable_ipv6**：是否为互联网网关启用IPv6，通过引用输入变量 internet_gateway_ipv6_enabled 进行赋值，默认值为`false`

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name              = "tf_test_vpc"
subnet_name           = "tf_test_subnet"
internet_gateway_name = "tf_test_internet_gateway"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建VPC互联网网关
4. 运行 `terraform show` 查看已创建的VPC互联网网关

## 参考信息

- [华为云弹性公网IP产品文档](https://support.huaweicloud.com/eip/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EIP VPC互联网网关最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eip/vpc-with-internet-gateway)
