# 部署WAF专业版Domain

## 应用场景

华为云Web应用防火墙（Web Application Firewall，WAF）专业版Domain是一种基于专属资源的Web安全防护服务，可以为指定的域名提供Web攻击防护。
通过配置WAF专业版Domain，可以为您的网站提供针对SQL注入、XSS跨站脚本、网页木马上传、命令注入等多种常见Web攻击的防护能力。
本最佳实践将介绍如何使用Terraform自动化部署一个WAF专业版Domain，包括创建WAF专业版实例、WAF策略和域名配置。

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
- [WAF专业版Domain资源（huaweicloud_waf_dedicated_domain）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_dedicated_domain)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── huaweicloud_vpc_subnet
    ├── data.huaweicloud_compute_flavors
    └── huaweicloud_waf_dedicated_instance

data.huaweicloud_compute_flavors
    └── huaweicloud_waf_dedicated_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_waf_dedicated_instance
        └── huaweicloud_waf_dedicated_domain

huaweicloud_networking_secgroup
    └── huaweicloud_waf_dedicated_instance

huaweicloud_waf_dedicated_instance
    └── huaweicloud_waf_policy
        └── huaweicloud_waf_dedicated_domain
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 前置资源准备

本最佳实践需要先创建VPC、子网、安全组和WAF专业版实例等前置资源。请按照[部署WAF专业版实例](dedicated_instance.md)最佳实践中的以下步骤进行准备：

- **步骤2**：创建VPC资源
- **步骤3**：通过数据源查询WAF实例资源创建所需的可用区
- **步骤4**：创建VPC子网资源
- **步骤5**：通过数据源查询WAF实例资源创建所需的计算规格
- **步骤6**：创建安全组资源
- **步骤7**：创建WAF专业版实例资源

完成上述步骤后，继续执行本最佳实践的后续步骤。

### 3. 创建WAF策略资源

在TF文件中添加以下脚本以告知Terraform创建WAF策略资源：

```hcl
variable "policy_name" {
  description = "The WAF policy name"
  type        = string
}

variable "policy_level" {
  description = "The WAF policy level"
  type        = number
  default     = 1
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建WAF策略资源，用于配置WAF专业版Domain的防护规则
resource "huaweicloud_waf_policy" "test" {
  name  = var.policy_name
  level = var.policy_level

  depends_on = [
    huaweicloud_waf_dedicated_instance.test
  ]
}
```

**参数说明**：
- **name**：WAF策略的名称，通过引用输入变量policy_name进行赋值
- **level**：WAF策略的防护级别，通过引用输入变量policy_level进行赋值，默认值为1
- **depends_on**：资源依赖关系，确保WAF专业版实例已创建

### 4. 创建WAF专业版Domain资源

在TF文件中添加以下脚本以告知Terraform创建WAF专业版Domain资源：

```hcl
variable "dedicated_mode_domain_name" {
  description = "The WAF dedicated mode domain name"
  type        = string
}

variable "dedicated_domain_client_protocol" {
  description = "The client protocol of the WAF dedicated domain"
  type        = string
  default     = "HTTP"
}

variable "dedicated_domain_server_protocol" {
  description = "The server protocol of the WAF dedicated domain"
  type        = string
  default     = "HTTP"
}

variable "dedicated_domain_address" {
  description = "The address of the WAF dedicated domain"
  type        = string
  default     = "192.168.0.14"
}

variable "dedicated_domain_port" {
  description = "The port of the WAF dedicated domain"
  type        = number
  default     = 8080
}

variable "dedicated_domain_type" {
  description = "The type of the WAF dedicated domain"
  type        = string
  default     = "ipv4"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建WAF专业版Domain资源
resource "huaweicloud_waf_dedicated_domain" "test" {
  domain      = var.dedicated_mode_domain_name
  policy_id   = huaweicloud_waf_policy.test.id
  keep_policy = true

  server {
    client_protocol = var.dedicated_domain_client_protocol
    server_protocol = var.dedicated_domain_server_protocol
    address         = var.dedicated_domain_address == "" ? cidrhost(huaweicloud_vpc_subnet.test.cidr, 4) : var.dedicated_domain_address
    port            = var.dedicated_domain_port
    type            = var.dedicated_domain_type
    vpc_id          = huaweicloud_vpc.test.id
  }
}
```

**参数说明**：
- **domain**：WAF专业版Domain的域名，通过引用输入变量dedicated_mode_domain_name进行赋值
- **policy_id**：WAF策略的ID，引用前面创建的WAF策略资源的ID
- **keep_policy**：是否保留策略，设置为true表示保留策略
- **server**：服务器配置块
  - **client_protocol**：客户端协议，通过引用输入变量dedicated_domain_client_protocol进行赋值，默认值为"HTTP"
  - **server_protocol**：服务器协议，通过引用输入变量dedicated_domain_server_protocol进行赋值，默认值为"HTTP"
  - **address**：服务器地址，如果dedicated_domain_address为空则使用cidrhost函数从子网网段中获取第4个IP地址，否则使用指定的dedicated_domain_address
  - **port**：服务器端口，通过引用输入变量dedicated_domain_port进行赋值，默认值为8080
  - **type**：服务器类型，通过引用输入变量dedicated_domain_type进行赋值，默认值为"ipv4"
  - **vpc_id**：VPC的ID，引用前面创建的VPC资源的ID

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC配置
vpc_name = "tf_test_vpc"
vpc_cidr = "192.168.0.0/16"

# 子网配置
subnet_name = "tf_test_subnet"

# WAF专业版实例配置
dedicated_instance_name               = "tf_test_waf_instance"
dedicated_instance_specification_code = "waf.instance.professional"
dedicated_instance_performance_type   = "normal"
dedicated_instance_cpu_core_count     = 4
dedicated_instance_memory_size        = 8

# 安全组配置
security_group_name = "tf_test_security_group"

# WAF策略配置
policy_name = "tf_test_waf_policy"
policy_level = 1

# WAF专业版Domain配置
dedicated_mode_domain_name = "www.example.com"
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

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建WAF专业版Domain
4. 运行 `terraform show` 查看已创建的WAF专业版Domain详情

## 参考信息

- [华为云WAF产品文档](https://support.huaweicloud.com/waf/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [WAF最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/waf/waf-dedicated-domain) 