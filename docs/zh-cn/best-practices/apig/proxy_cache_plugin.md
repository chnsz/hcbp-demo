# 创建proxy_cache插件

## 应用场景

API网关（API Gateway）的proxy_cache插件是一种用于缓存API响应数据的插件，可以显著提升API的响应速度和降低后端服务的负载。proxy_cache插件支持配置缓存键、缓存策略、TTL（Time To Live）等参数，帮助您实现灵活的API响应缓存管理。通过合理配置缓存策略，可以减少重复请求对后端服务的压力，提高整体系统的性能和可用性。本最佳实践将介绍如何使用Terraform自动化创建API网关的proxy_cache插件。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [API网关实例资源（huaweicloud_apig_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_instance)
- [API网关插件资源（huaweicloud_apig_plugin）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_plugin)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_apig_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_apig_instance

huaweicloud_networking_secgroup
    └── huaweicloud_apig_instance

huaweicloud_apig_instance
    └── huaweicloud_apig_plugin
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询API网关实例资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建API网关实例：

```hcl
variable "availability_zones" {
  description = "实例所属的可用区列表"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "availability_zones_count" {
  description = "实例所属的可用区数量"
  type        = number
  default     = 1
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建API网关实例
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zones` 为空时创建数据源（即执行可用区列表查询）

### 3. 创建VPC资源

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署API网关实例
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量vpc_cidr进行赋值

### 4. 创建VPC子网资源

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

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
  description = "子网的网关IP地址"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署API网关实例
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网网段，当subnet_cidr为空时，自动从VPC的CIDR中计算子网网段；否则使用输入变量subnet_cidr的值
- **gateway_ip**：网关IP地址，当subnet_gateway_ip为空时，自动从计算的子网网段中获取网关IP；否则使用输入变量subnet_gateway_ip的值

### 5. 创建安全组资源

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "安全组名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署API网关实例
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 6. 创建API网关实例资源

在TF文件中添加以下脚本以告知Terraform创建API网关实例资源：

```hcl
variable "instance_name" {
  description = "API网关实例名称"
  type        = string
}

variable "instance_edition" {
  description = "API网关实例规格"
  type        = string
  default     = "BASIC"
}

variable "enterprise_project_id" {
  description = "企业项目ID"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建API网关实例资源
resource "huaweicloud_apig_instance" "test" {
  name                  = var.instance_name
  edition               = var.instance_edition
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zones    = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, var.availability_zones_count), null) : var.availability_zones
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **name**：API网关实例名称，通过引用输入变量instance_name进行赋值
- **edition**：实例规格，通过引用输入变量instance_edition进行赋值，默认值为"BASIC"
- **vpc_id**：VPC的ID，引用前面创建的VPC资源的ID
- **subnet_id**：子网的ID，引用前面创建的子网资源的ID
- **security_group_id**：安全组的ID，引用前面创建的安全组资源的ID
- **availability_zones**：可用区列表，当availability_zones为空时，使用可用区列表查询数据源的结果；否则使用输入变量availability_zones的值
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值

### 7. 创建API网关插件资源

在TF文件中添加以下脚本以告知Terraform创建API网关插件资源：

```hcl
variable "plugin_name" {
  description = "API网关插件名称"
  type        = string
}

variable "plugin_description" {
  description = "API网关插件描述"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建API网关插件资源
resource "huaweicloud_apig_plugin" "test" {
  instance_id = huaweicloud_apig_instance.test.id
  name        = var.plugin_name
  type        = "proxy_cache"
  description = var.plugin_description

  content = jsonencode({
    cache_key = {
      system_params = []
      parameters = [
        "custom_param"
      ],
      headers = []
    },
    cache_http_status_and_ttl = [
      {
        http_status = [
          202,
          203
        ],
        ttl = 5
      }
    ],
    client_cache_control = {
      mode  = "off",
      datas = []
    },
    cacheable_headers = [
      "X-Custom-Header"
    ]
  })
}
```

**参数说明**：
- **instance_id**：API网关实例的ID，引用前面创建的API网关实例资源的ID
- **name**：插件名称，通过引用输入变量plugin_name进行赋值
- **type**：插件类型，设置为"proxy_cache"表示创建代理缓存插件
- **description**：插件描述，通过引用输入变量plugin_description进行赋值
- **content**：插件配置内容，使用JSON格式编码，包含以下配置项：
  - **cache_key**：缓存键配置，包含系统参数、自定义参数和请求头
  - **cache_http_status_and_ttl**：缓存HTTP状态码和TTL配置，指定哪些HTTP状态码的响应需要缓存以及缓存时间
  - **client_cache_control**：客户端缓存控制配置，设置客户端缓存模式
  - **cacheable_headers**：可缓存的请求头列表

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`terraform.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC配置
vpc_name              = "tf_test_apig_instance_vpc"

# 子网配置
subnet_name           = "tf_test_apig_instance_subnet"

# 安全组配置
security_group_name   = "tf_test_apig_instance_security_group"

# API网关实例配置
instance_name         = "tf_test_apig_instance"
enterprise_project_id = "0"

# API网关插件配置
plugin_name           = "tf_test_apig_plugin"
plugin_description    = "Created by Terraform script"
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

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建proxy_cache插件
4. 运行 `terraform show` 查看已创建的proxy_cache插件详情

## 参考信息

- [华为云API网关产品文档](https://support.huaweicloud.com/apig/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [APIG proxy_cache插件最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/apig/proxy-cache-plugin)
