# 部署WAF专业版实例

## 应用场景

华为云Web应用防火墙（Web Application Firewall，WAF）专业版是一种基于专属资源的Web安全防护服务，可以有效防御SQL注入、XSS跨站脚本、网页木马上传、命令注入等多种常见Web攻击。
专业版WAF实例提供独享的资源，为企业提供更加定制化的Web安全防护，适合对Web应用安全防护有高性能要求、对合规性和数据隔离有严格要求的场景。
本最佳实践将介绍如何使用Terraform自动化部署一个WAF专业版实例。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [计算规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [WAF专业版实例资源（huaweicloud_waf_dedicated_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_dedicated_instance)
- [WAF策略资源（huaweicloud_waf_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_policy)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_waf_dedicated_instance

data.huaweicloud_compute_flavors
    └── huaweicloud_waf_dedicated_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_waf_dedicated_instance

huaweicloud_networking_secgroup
    └── huaweicloud_waf_dedicated_instance

huaweicloud_waf_dedicated_instance
    └── huaweicloud_waf_policy
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询WAF实例资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建WAF专业版实例：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建WAF专业版实例
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 此数据源无需配置参数，会默认获取当前区域下所有可用区信息。

### 3. 通过数据源查询WAF实例资源创建所需的计算规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的计算规格：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的计算规格信息，用于创建WAF专业版实例
data "huaweicloud_compute_flavors" "test" {
  availability_zone = data.huaweicloud_availability_zones.test.names[0]
  performance_type  = "normal"
  cpu_core_count    = 2
  memory_size       = 4
}
```

**参数说明**：
- **availability_zone**：计算规格所在的可用区，使用可用区列表查询数据源的第一个可用区
- **performance_type**：性能类型，使用"normal"表示通用型
- **cpu_core_count**：CPU核数，设置为2核
- **memory_size**：内存大小（GB），设置为4GB

### 4. 创建VPC资源

在TF文件中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
}

variable "enterprise_project_id" {
  description = "企业项目ID"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署WAF专业版实例
resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量vpc_cidr进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值

### 5. 创建VPC子网资源

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "子网名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署WAF专业版实例
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  availability_zone = data.huaweicloud_availability_zones.test.names[0]
  name              = var.subnet_name
  cidr              = cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip        = cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID
- **availability_zone**：子网所在的可用区，使用可用区列表查询数据源的第一个可用区
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR网段，使用cidrsubnet函数从VPC的CIDR网段中划分一个子网段
- **gateway_ip**：子网的网关IP，使用cidrhost函数从子网网段中获取第一个IP地址作为网关IP

### 6. 创建安全组资源

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "安全组名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署WAF专业版实例
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组的名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 7. 创建WAF专业版实例资源

在TF文件中添加以下脚本以告知Terraform创建WAF专业版实例资源：

```hcl
variable "waf_dedicated_instance_name" {
  description = "WAF专业版实例名称"
  type        = string
}

variable "waf_dedicated_instance_specification_code" {
  description = "WAF专业版实例规格编码"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建WAF专业版实例资源
resource "huaweicloud_waf_dedicated_instance" "test" {
  name                  = var.waf_dedicated_instance_name
  available_zone        = data.huaweicloud_availability_zones.test.names[0]
  specification_code    = var.waf_dedicated_instance_specification_code
  ecs_flavor            = data.huaweicloud_compute_flavors.test.flavors[0].id
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  enterprise_project_id = var.enterprise_project_id

  security_group = [
    huaweicloud_networking_secgroup.test.id
  ]
}
```

**参数说明**：
- **name**：WAF专业版实例的名称，通过引用输入变量waf_dedicated_instance_name进行赋值
- **available_zone**：WAF专业版实例所在的可用区，使用可用区列表查询数据源的第一个可用区
- **specification_code**：WAF专业版实例规格编码，通过引用输入变量waf_dedicated_instance_specification_code进行赋值
- **ecs_flavor**：WAF专业版实例使用的计算规格，使用计算规格列表查询数据源的第一个规格
- **vpc_id**：WAF专业版实例所在的VPC的ID，引用前面创建的VPC资源的ID
- **subnet_id**：WAF专业版实例所在的子网的ID，引用前面创建的子网资源的ID
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值
- **security_group**：WAF专业版实例使用的安全组ID列表，引用前面创建的安全组资源的ID

### 8. 创建WAF策略资源（huaweicloud_waf_policy）

在TF文件中添加以下脚本以告知Terraform创建WAF策略资源：

```hcl
variable "waf_policy_name" {
  description = "WAF策略名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建WAF策略资源，用于配置WAF专业版实例的防护规则
resource "huaweicloud_waf_policy" "test" {
  name  = var.waf_policy_name
  level = 1

  # 确保WAF专业版实例已创建
  depends_on = [huaweicloud_waf_dedicated_instance.test]
}
```

**参数说明**：
- **name**：WAF策略的名称，通过引用输入变量waf_policy_name进行赋值
- **level**：WAF策略的防护级别，设置为1表示低级别防护
- **depends_on**：资源依赖关系，确保WAF专业版实例已创建

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 企业项目
enterprise_project_id = "0"

# VPC配置
vpc_name            = "tf_test_vpc"
vpc_cidr            = "192.168.0.0/16"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_secgroup"

# WAF配置
waf_dedicated_instance_name               = "tf_test_waf_dedicated_instance"
waf_dedicated_instance_specification_code = "waf.instance.professional"
waf_policy_name                           = "tf_test_waf_policy"
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

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建WAF专业版实例
4. 运行 `terraform show` 查看已创建的WAF专业版实例详情

## 参考信息

- [华为云WAF产品文档](https://support.huaweicloud.com/waf/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [WAF最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/waf/waf-dedicated-instance) 