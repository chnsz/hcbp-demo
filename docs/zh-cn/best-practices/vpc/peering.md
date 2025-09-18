# 部署对等连接

## 应用场景

VPC Peering（对等连接）用于实现两个虚拟私有云（VPC）之间的私有网络互通，适用于多VPC组网、跨业务系统通信等场景。通过VPC Peering，用户可以在不同VPC之间实现安全、低延迟的内网通信，满足企业多网络环境下的资源互访需求。本最佳实践将介绍如何使用Terraform自动化部署两个VPC及其子网，并建立VPC Peering连接和路由。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [VPC Peering连接资源（huaweicloud_vpc_peering_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_peering_connection)
- [VPC路由资源（huaweicloud_vpc_route）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_route)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
    └── huaweicloud_vpc_peering_connection
        └── huaweicloud_vpc_route
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区信息

在TF文件（如main.tf）中添加以下脚本以查询可用区信息（本实践未直接用到，但建议保留以便后续扩展）：

```hcl
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 此数据源无需配置参数，会默认获取当前区域下所有可用区信息。

### 3. 创建VPC及子网资源

在TF文件中添加以下脚本以告知Terraform批量创建VPC及其子网：

```hcl
variable "vpc_configurations" {
  description = "The list of VPC configurations for peering connection"
  type = list(object({
    vpc_name              = string
    vpc_cidr              = string
    subnet_name           = string
    enterprise_project_id = optional(string, null)
  }))
  validation {
    condition     = length(var.vpc_configurations) == 2
    error_message = "Exactly 2 VPC configurations are required for peering connection."
  }
}

# 创建VPC资源
resource "huaweicloud_vpc" "test" {
  count = length(var.vpc_configurations)
  name                  = lookup(var.vpc_configurations[count.index], "vpc_name", null)
  cidr                  = lookup(var.vpc_configurations[count.index], "vpc_cidr", null)
  enterprise_project_id = lookup(var.vpc_configurations[count.index], "enterprise_project_id", null)
}

# 创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  count = length(var.vpc_configurations)
  vpc_id     = huaweicloud_vpc.test[count.index].id
  name       = lookup(var.vpc_configurations[count.index], "subnet_name", null)
  cidr       = try(cidrsubnet(lookup(var.vpc_configurations[count.index], "vpc_cidr", null), 6, 32), null)
  gateway_ip = try(cidrhost(cidrsubnet(lookup(var.vpc_configurations[count.index], "vpc_cidr", null), 6, 32), 1), null)
}
```

**参数说明**：
- **vpc_configurations**：VPC及子网的配置列表，需包含2组配置
- **name/cidr/enterprise_project_id**：VPC名称、网段、企业项目ID，分别通过vpc_configurations赋值
- **subnet_name/cidr/gateway_ip**：子网名称、网段、网关IP，分别通过vpc_configurations和计算函数赋值

### 4. 创建VPC Peering连接

在TF文件中添加以下脚本以告知Terraform创建VPC Peering连接：

```hcl
variable "peering_connection_name" {
  description = "The name of the VPC peering connection"
  type        = string
}

# 创建VPC Peering连接
resource "huaweicloud_vpc_peering_connection" "test" {
  count = length(var.vpc_configurations) == 2 ? 1 : 0
  name        = var.peering_connection_name
  vpc_id      = try(huaweicloud_vpc.test[0].id, null) # source VPC
  peer_vpc_id = try(huaweicloud_vpc.test[1].id, null) # target VPC
}
```

**参数说明**：
- **name**：Peering连接名称，通过输入变量peering_connection_name赋值
- **vpc_id/peer_vpc_id**：源VPC和目标VPC的ID，分别引用前面创建的VPC资源

### 5. 配置VPC Peering路由

在TF文件中添加以下脚本以告知Terraform为每个VPC配置Peering路由：

```hcl
# 配置VPC Peering路由
resource "huaweicloud_vpc_route" "test" {
  count = length(var.vpc_configurations)
  vpc_id      = huaweicloud_vpc.test[count.index].id
  destination = try(cidrsubnet(lookup(var.vpc_configurations[count.index], "vpc_cidr", null), 6, 33), null)
  type        = "peering"
  nexthop     = try(huaweicloud_vpc_peering_connection.test[0].id, null)
}
```

**参数说明**：
- **vpc_id**：路由所属的VPC的ID，引用前面创建的VPC资源
- **destination**：对端VPC的网段，自动计算
- **type**：路由类型，固定为"peering"
- **nexthop**：下一跳，引用前面创建的VPC Peering连接的ID

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC Peering配置
vpc_configurations = [
  {
    vpc_name    = "tf_test_source_vpc"
    vpc_cidr    = "192.168.0.0/18"
    subnet_name = "tf_test_source_subnet"
  },
  {
    vpc_name    = "tf_test_target_vpc"
    vpc_cidr    = "192.168.128.0/18"
    subnet_name = "tf_test_target_subnet"
  }
]
peering_connection_name = "tf_test_peering"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="peering_connection_name=my-peering"`
2. 环境变量：`export TF_VAR_peering_connection_name=my-peering`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建VPC Peering连接及相关资源
4. 运行 `terraform show` 查看已创建的VPC Peering连接及路由详情

## 参考信息

- [华为云VPC产品文档](https://support.huaweicloud.com/vpc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VPC Peering最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpc)
