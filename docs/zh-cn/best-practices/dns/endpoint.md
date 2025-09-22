# 部署终端节点

## 应用场景

云解析服务（Domain Name Service, DNS）是华为云提供的高可用、高性能的域名解析服务，支持公网域名解析和私网域名解析。DNS服务提供智能解析、负载均衡、健康检查等功能，帮助用户实现域名的智能调度和故障转移。

DNS终端节点是DNS服务中的网络接入点，用于在VPC网络中提供DNS解析服务。通过终端节点，企业可以在私有网络中部署DNS服务，实现内网域名解析、私网DNS转发等功能。终端节点支持入站和出站两种方向，满足不同场景的DNS解析需求。本最佳实践将介绍如何使用Terraform自动化部署DNS终端节点，包括VPC创建、子网配置和终端节点部署。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [DNS终端节点资源（huaweicloud_dns_endpoint）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_endpoint)

### 资源/数据源依赖关系

```
huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_dns_endpoint.test
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

### 3. 创建VPC子网

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "子网名称"
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：VPC ID，通过引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网CIDR块，优先使用输入变量，如果为空则通过cidrsubnet函数计算
- **gateway_ip**：网关IP，优先使用输入变量，如果为空则通过cidrhost函数计算

### 4. 创建DNS终端节点

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DNS终端节点资源：

```hcl
variable "dns_endpoint_name" {
  description = "DNS终端节点名称"
  type        = string
}

variable "dns_endpoint_direction" {
  description = "DNS终端节点方向"
  type        = string
  default     = "inbound"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DNS终端节点资源
resource "huaweicloud_dns_endpoint" "test" {
  name      = var.dns_endpoint_name
  direction = var.dns_endpoint_direction

  ip_addresses {
    subnet_id = huaweicloud_vpc_subnet.test.id
  }

  ip_addresses {
    subnet_id = huaweicloud_vpc_subnet.test.id
  }
}
```

**参数说明**：
- **name**：终端节点名称，通过引用输入变量dns_endpoint_name进行赋值
- **direction**：终端节点方向，通过引用输入变量dns_endpoint_direction进行赋值，默认为"inbound"（入站）
- **ip_addresses.subnet_id**：IP地址子网ID，通过引用VPC子网资源（huaweicloud_vpc_subnet.test）的ID进行赋值

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 网络配置
vpc_name    = "tf_test_dns_endpoint"
subnet_name = "tf_test_dns_endpoint"

# DNS终端节点配置
dns_endpoint_name = "tf_test_dns_endpoint"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="dns_endpoint_name=my-endpoint"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DNS终端节点
4. 运行 `terraform show` 查看已创建的DNS终端节点

## 参考信息

- [华为云DNS产品文档](https://support.huaweicloud.com/dns/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DNS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dns)
