# 部署基础VPC网络

## 应用场景

虚拟私有云（Virtual Private Cloud，VPC）是用户在华为云上自定义和管理的逻辑隔离网络空间。通过VPC，用户可以灵活划分子网、配置路由和安全策略，实现云上资源的安全隔离和高效管理。本最佳实践将介绍如何使用Terraform自动化部署一个基础的VPC及其子网。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)

### 资源/数据源依赖关系

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "172.16.0.0/16"
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the VPC belongs"
  type        = string
  default     = null
}

# 创建VPC资源
resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量vpc_cidr进行赋值，默认值为"172.16.0.0/16"
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null

### 3. 创建VPC子网资源

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "172.16.10.0/24"
}

variable "subnet_gateway" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = "172.16.10.1"
}

variable "dns_list" {
  description = "The list of DNS server IP addresses"
  type        = list(string)
  default     = null
}

# 创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway
  dns_list   = var.dns_list
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR网段，通过引用输入变量subnet_cidr进行赋值，默认值为"172.16.10.0/24"
- **gateway_ip**：子网的网关IP，通过引用输入变量subnet_gateway进行赋值，默认值为"172.16.10.1"
- **dns_list**：子网的DNS服务器IP地址列表，通过引用输入变量dns_list进行赋值，默认值为null

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC配置
vpc_name              = "tf_test_vpc"
vpc_cidr              = "172.16.0.0/16"
enterprise_project_id = null

# 子网配置
subnet_name    = "tf_test_subnet"
subnet_cidr    = "172.16.10.0/24"
subnet_gateway = "172.16.10.1"
dns_list       = ["114.114.114.114", "8.8.8.8"]
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

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建VPC及子网
4. 运行 `terraform show` 查看已创建的VPC及子网详情

## 参考信息

- [华为云VPC产品文档](https://support.huaweicloud.com/vpc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VPC最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpc) 