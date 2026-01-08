# 部署服务

## 应用场景

云容器实例（Cloud Container Instance，CCI）服务是CCI服务提供的服务发现和负载均衡功能，用于为容器应用提供统一的访问入口。通过创建CCI服务，您可以将容器应用暴露给外部访问，实现服务的负载均衡和流量分发。通过Terraform自动化创建CCI服务，可以确保服务配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建CCI服务，包括VPC、子网、安全组、命名空间和ELB负载均衡器的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCI命名空间资源（huaweicloud_cciv2_namespace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_namespace)
- [ELB负载均衡器资源（huaweicloud_elb_loadbalancer）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/elb_loadbalancer)
- [CCI服务资源（huaweicloud_cciv2_service）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_service)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_elb_loadbalancer

huaweicloud_networking_secgroup
    └── huaweicloud_cciv2_service

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_elb_loadbalancer
        └── huaweicloud_cciv2_service

huaweicloud_cciv2_namespace
    ├── huaweicloud_elb_loadbalancer
    └── huaweicloud_cciv2_service

huaweicloud_elb_loadbalancer
    └── huaweicloud_cciv2_service
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ELB负载均衡器：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建ELB负载均衡器
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
此数据源无需额外参数，默认查询当前区域下所有可用的可用区信息。

### 3. 创建安全组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "The name of security group"
  type        = string
  default     = "tf-test-secgroup"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值，默认值为"tf-test-secgroup"
- **delete_default_rules**：是否删除默认规则，设置为true

### 4. 创建VPC资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "The name of VPC"
  type        = string
  default     = "tf-test-vpc"
}

variable "vpc_cidr" {
  description = "The CIDR block of VPC"
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
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值，默认值为"tf-test-vpc"
- **cidr**：VPC的CIDR网段，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"

### 5. 创建VPC子网资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "The name of subnet"
  type        = string
  default     = "tf-test-subnet"
}

variable "subnet_cidr" {
  description = "The CIDR block of subnet"
  type        = string
  default     = "192.168.0.0/24"
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of subnet"
  type        = string
  default     = "192.168.0.1"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源（huaweicloud_vpc.test）的ID
- **name**：子网名称，通过引用输入变量subnet_name进行赋值，默认值为"tf-test-subnet"
- **cidr**：子网的CIDR网段，通过引用输入变量subnet_cidr进行赋值，默认值为"192.168.0.0/24"
- **gateway_ip**：子网的网关IP，通过引用输入变量subnet_gateway_ip进行赋值，默认值为"192.168.0.1"

### 6. 创建CCI命名空间资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CCI命名空间资源：

```hcl
variable "namespace_name" {
  description = "The name of CCI namespace"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCI命名空间资源
resource "huaweicloud_cciv2_namespace" "test" {
  name = var.namespace_name
}
```

**参数说明**：
- **name**：命名空间名称，通过引用输入变量namespace_name进行赋值

### 7. 创建ELB负载均衡器资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ELB负载均衡器资源：

```hcl
variable "elb_name" {
  description = "The name of ELB load balancer"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ELB负载均衡器资源
resource "huaweicloud_elb_loadbalancer" "test" {
  depends_on = [huaweicloud_cciv2_namespace.test]

  name              = var.elb_name
  cross_vpc_backend = true
  vpc_id            = huaweicloud_vpc.test.id
  ipv4_subnet_id    = huaweicloud_vpc_subnet.test.ipv4_subnet_id

  availability_zone = [
    try(data.huaweicloud_availability_zones.test.names[1], "")
  ]
}
```

**参数说明**：
- **depends_on**：显式依赖关系，确保命名空间资源先于ELB负载均衡器资源创建
- **name**：负载均衡器名称，通过引用输入变量elb_name进行赋值
- **cross_vpc_backend**：是否支持跨VPC后端，设置为true
- **vpc_id**：VPC ID，引用前面创建的VPC资源（huaweicloud_vpc.test）的ID
- **ipv4_subnet_id**：IPv4子网ID，引用前面创建的VPC子网资源（huaweicloud_vpc_subnet.test）的IPv4子网ID
- **availability_zone**：可用区列表，根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值，使用第二个可用区

### 8. 创建CCI服务资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CCI服务资源：

```hcl
variable "service_name" {
  description = "The name of CCI service"
  type        = string
}

variable "selector_app" {
  description = "The app label of selector"
  type        = string
  default     = "test1"
}

variable "service_type" {
  description = "The type of service"
  type        = string
  default     = "LoadBalancer"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCI服务资源
resource "huaweicloud_cciv2_service" "test" {
  depends_on = [huaweicloud_elb_loadbalancer.test]

  namespace = huaweicloud_cciv2_namespace.test.name
  name      = var.service_name

  annotations = {
    "kubernetes.io/elb.class" = "elb"
    "kubernetes.io/elb.id"    = huaweicloud_elb_loadbalancer.test.id
  }

  ports {
    name         = "test"
    app_protocol = "TCP"
    protocol     = "TCP"
    port         = 87
    target_port  = 65529
  }

  selector = {
    app = var.selector_app
  }

  type = var.service_type

  lifecycle {
    ignore_changes = [
      annotations,
    ]
  }
}
```

**参数说明**：
- **depends_on**：显式依赖关系，确保ELB负载均衡器资源先于CCI服务资源创建
- **namespace**：命名空间名称，引用前面创建的CCI命名空间资源（huaweicloud_cciv2_namespace.test）的名称
- **name**：服务名称，通过引用输入变量service_name进行赋值
- **annotations.kubernetes.io/elb.class**：ELB类型注解，设置为"elb"
- **annotations.kubernetes.io/elb.id**：ELB ID注解，引用前面创建的ELB负载均衡器资源（huaweicloud_elb_loadbalancer.test）的ID
- **ports.name**：端口名称，设置为"test"
- **ports.app_protocol**：应用协议，设置为"TCP"
- **ports.protocol**：协议类型，设置为"TCP"
- **ports.port**：服务端口，设置为87
- **ports.target_port**：目标端口，设置为65529
- **selector.app**：选择器应用标签，通过引用输入变量selector_app进行赋值，默认值为"test1"
- **type**：服务类型，通过引用输入变量service_type进行赋值，默认值为"LoadBalancer"

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# CCI服务配置
elb_name       = "tf-test-elb"
service_name   = "tf-test-service"
namespace_name = "tf-test-namespace"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="service_name=test-service" -var="namespace_name=test-namespace"`
2. 环境变量：`export TF_VAR_service_name=test-service` 和 `export TF_VAR_namespace_name=test-namespace`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建CCI服务：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CCI服务
4. 运行 `terraform show` 查看已创建的CCI服务详情

> 注意：服务必须创建在已存在的命名空间中。选择器标签必须与Pod标签匹配。ELB负载均衡器必须提前创建。

## 参考信息

- [华为云CCI产品文档](https://support.huaweicloud.com/cci/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [服务最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cci/service)
