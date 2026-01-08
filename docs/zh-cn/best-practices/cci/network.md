# 部署网络

## 应用场景

云容器实例（Cloud Container Instance，CCI）网络是CCI服务提供的网络配置功能，用于为容器应用提供网络连接能力。通过创建CCI网络，您可以将容器应用连接到VPC网络，实现容器与云上其他资源的互通。通过Terraform自动化创建CCI网络，可以确保网络配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建CCI网络，包括VPC、子网、安全组和命名空间的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCI命名空间资源（huaweicloud_cciv2_namespace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_namespace)
- [CCI网络资源（huaweicloud_cciv2_network）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_network)

### 资源/数据源依赖关系

```
huaweicloud_networking_secgroup
    └── huaweicloud_cciv2_network

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_cciv2_network

huaweicloud_cciv2_namespace
    └── huaweicloud_cciv2_network
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建安全组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "The name of the security group"
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

### 3. 创建VPC资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
  default     = "tf-test-vpc"
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
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

### 4. 创建VPC子网资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
  default     = "tf-test-subnet"
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "192.168.0.0/24"
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
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

### 5. 创建CCI命名空间资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CCI命名空间资源：

```hcl
variable "namespace_name" {
  description = "The name of the CCI namespace"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCI命名空间资源
resource "huaweicloud_cciv2_namespace" "test" {
  name = var.namespace_name
}
```

**参数说明**：
- **name**：命名空间名称，通过引用输入变量namespace_name进行赋值

### 6. 创建CCI网络资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CCI网络资源：

```hcl
variable "network_name" {
  description = "The name of the CCI network"
  type        = string
}

variable "warm_pool_size" {
  description = "The size of the warm pool for the network"
  type        = string
  default     = "10"
}

variable "warm_pool_recycle_interval" {
  description = "The recycle interval of the warm pool in hours"
  type        = string
  default     = "2"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCI网络资源
resource "huaweicloud_cciv2_network" "test" {
  depends_on = [huaweicloud_cciv2_namespace.test]

  namespace = huaweicloud_cciv2_namespace.test.name
  name      = var.network_name

  annotations = {
    "yangtse.io/project-id"                 = huaweicloud_cciv2_namespace.test.annotations["tenant.kubernetes.io/project-id"]
    "yangtse.io/domain-id"                  = huaweicloud_cciv2_namespace.test.annotations["tenant.kubernetes.io/domain-id"]
    "yangtse.io/warm-pool-size"             = var.warm_pool_size
    "yangtse.io/warm-pool-recycle-interval" = var.warm_pool_recycle_interval
  }

  subnets {
    subnet_id = huaweicloud_vpc_subnet.test.ipv4_subnet_id
  }

  security_group_ids = [huaweicloud_networking_secgroup.test.id]
}
```

**参数说明**：
- **depends_on**：显式依赖关系，确保命名空间资源先于网络资源创建
- **namespace**：命名空间名称，引用前面创建的CCI命名空间资源（huaweicloud_cciv2_namespace.test）的名称
- **name**：网络名称，通过引用输入变量network_name进行赋值
- **annotations.yangtse.io/project-id**：项目ID注解，从命名空间的注解中自动继承
- **annotations.yangtse.io/domain-id**：域ID注解，从命名空间的注解中自动继承
- **annotations.yangtse.io/warm-pool-size**：预热池大小注解，通过引用输入变量warm_pool_size进行赋值，默认值为"10"
- **annotations.yangtse.io/warm-pool-recycle-interval**：预热池回收间隔注解（小时），通过引用输入变量warm_pool_recycle_interval进行赋值，默认值为"2"
- **subnets.subnet_id**：子网ID，引用前面创建的VPC子网资源（huaweicloud_vpc_subnet.test）的IPv4子网ID
- **security_group_ids**：安全组ID列表，引用前面创建的安全组资源（huaweicloud_networking_secgroup.test）的ID

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# CCI网络配置
network_name   = "tf-test-network"
namespace_name = "tf-test-namespace"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="network_name=test-network" -var="namespace_name=test-namespace"`
2. 环境变量：`export TF_VAR_network_name=test-network` 和 `export TF_VAR_namespace_name=test-namespace`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建CCI网络：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CCI网络
4. 运行 `terraform show` 查看已创建的CCI网络详情

> 注意：网络必须关联至少一个子网。预热池配置有助于通过预分配资源来减少Pod启动时间。网络名称在命名空间内必须唯一。注解yangtse.io/project-id和yangtse.io/domain-id会自动从命名空间继承。

## 参考信息

- [华为云CCI产品文档](https://support.huaweicloud.com/cci/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [网络最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cci/network)
