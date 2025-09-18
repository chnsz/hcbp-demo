# 部署带有自定义认证的API

## 应用场景

华为云API网关（APIG）支持多种认证方式，包括IAM认证、APP认证等。同时，API网关也支持用户使用自己的认证方式（以下简称自定义认证），以便更好地兼容已有业务能力。自定义认证可以满足复杂的认证需求，如集成第三方认证系统、实现自定义的权限控制逻辑等。本最佳实践将介绍如何使用Terraform自动化部署带有自定义认证的API以及如何使用FunctionGraph函数实现API的前端认证。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [API网关实例资源（huaweicloud_apig_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_instance)
- [API网关分组资源（huaweicloud_apig_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_group)
- [API定义资源（huaweicloud_apig_api）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_api)
- [自定义认证器资源（huaweicloud_apig_custom_authorizer）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_custom_authorizer)
- [函数资源（huaweicloud_fgs_function）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/fgs_function)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_apig_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_apig_instance

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
        └── huaweicloud_apig_instance

huaweicloud_apig_instance
    └── huaweicloud_apig_group
        └── huaweicloud_apig_api
            └── huaweicloud_apig_custom_authorizer
                └── huaweicloud_fgs_function
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询API网关实例资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建API网关实例：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建API网关实例
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
此数据源无需额外参数，默认查询当前区域下所有可用的可用区信息。

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
}

variable "subnet_gateway" {
  description = "子网的网关地址"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署API网关实例
resource "huaweicloud_vpc_subnet" "test" {
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway
  vpc_id     = huaweicloud_vpc.test.id
}
```

**参数说明**：
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网网段，通过引用输入变量subnet_cidr进行赋值
- **gateway_ip**：网关IP地址，通过引用输入变量subnet_gateway进行赋值
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID

### 5. 创建安全组资源

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "secgroup_name" {
  description = "安全组名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署API网关实例
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.secgroup_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量secgroup_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 6. 创建安全组规则资源

在TF文件中添加以下脚本以告知Terraform创建安全组规则资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组规则资源，用于配置API网关实例的访问控制
resource "huaweicloud_networking_secgroup_rule" "allow_web" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype        = "IPv4"
  protocol         = "tcp"
  port_range_min   = 80
  port_range_max   = 80
  remote_ip_prefix = "0.0.0.0/0"
  description      = "允许HTTP访问"
}

resource "huaweicloud_networking_secgroup_rule" "allow_https" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype        = "IPv4"
  protocol         = "tcp"
  port_range_min   = 443
  port_range_max   = 443
  remote_ip_prefix = "0.0.0.0/0"
  description      = "允许HTTPS访问"
}
```

**参数说明**：
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID
- **direction**：规则方向，设置为"ingress"表示入方向规则
- **ethertype**：网络协议版本，设置为"IPv4"
- **protocol**：协议类型，设置为"tcp"
- **port_range_min/port_range_max**：端口范围，分别设置为80和443
- **remote_ip_prefix**：允许访问的IP范围，设置为"0.0.0.0/0"表示允许所有IP访问
- **description**：规则描述

### 7. 创建API网关实例资源

在TF文件中添加以下脚本以告知Terraform创建API网关实例资源：

```hcl
variable "apig_name" {
  description = "API网关实例名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建API网关实例资源
resource "huaweicloud_apig_instance" "test" {
  name                  = var.apig_name
  edition              = "BASIC"
  vpc_id               = huaweicloud_vpc.test.id
  subnet_id            = huaweicloud_vpc_subnet.test.id
  security_group_id    = huaweicloud_networking_secgroup.test.id
  availability_zones   = [data.huaweicloud_availability_zones.test.names[0]]
  description          = "API网关实例"
  enterprise_project_id = "0"
}
```

**参数说明**：
- **name**：API网关实例名称，通过引用输入变量apig_name进行赋值
- **edition**：实例规格，设置为"BASIC"
- **vpc_id**：VPC的ID，引用前面创建的VPC资源的ID
- **subnet_id**：子网的ID，引用前面创建的子网资源的ID
- **security_group_id**：安全组的ID，引用前面创建的安全组资源的ID
- **availability_zones**：可用区列表，使用可用区列表查询数据源的第一个可用区
- **description**：实例描述
- **enterprise_project_id**：企业项目ID，设置为"0"

### 8. 创建API网关分组资源

在TF文件中添加以下脚本以告知Terraform创建API网关分组资源：

```hcl
variable "group_name" {
  description = "API分组名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建API网关分组资源
resource "huaweicloud_apig_group" "test" {
  name        = var.group_name
  description = "API分组"
  instance_id = huaweicloud_apig_instance.test.id
}
```

**参数说明**：
- **name**：API分组名称，通过引用输入变量group_name进行赋值
- **description**：分组描述
- **instance_id**：API网关实例的ID，引用前面创建的API网关实例资源的ID

### 9. 创建函数资源

在TF文件中添加以下脚本以告知Terraform创建函数资源：

```hcl
variable "function_name" {
  description = "函数名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建函数资源
resource "huaweicloud_fgs_function" "test" {
  name        = var.function_name
  app         = "default"
  handler     = "index.handler"
  memory_size = 128
  timeout     = 30
  runtime     = "Python3.6"
  code_type   = "inline"
  
  func_code = <<EOF
import json

def handler(event, context):
    # 获取认证头信息
    token = event['headers'].get('Authorization', '')
    
    if not token:
        return {
            'statusCode': 401,
            'body': json.dumps({
                'message': 'Unauthorized'
            })
        }
    
    # 在这里实现您的认证逻辑
    # 例如：验证token、检查用户权限等
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'principalId': 'user123',
            'context': {
                'userId': 'user123',
                'userRole': 'admin'
            }
        })
    }
EOF
}
```

**参数说明**：
- **name**：函数名称，通过引用输入变量function_name进行赋值
- **app**：函数所属应用，设置为"default"
- **handler**：函数入口，设置为"index.handler"
- **memory_size**：函数运行时内存大小，设置为128MB
- **timeout**：函数超时时间，设置为30秒
- **runtime**：运行时环境，设置为"Python3.6"
- **code_type**：代码类型，设置为"inline"
- **func_code**：函数代码，包含自定义认证逻辑

### 10. 创建自定义认证器资源

在TF文件中添加以下脚本以告知Terraform创建自定义认证器资源：

```hcl
variable "authorizer_name" {
  description = "自定义认证器名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建自定义认证器资源
resource "huaweicloud_apig_custom_authorizer" "test" {
  instance_id  = huaweicloud_apig_instance.test.id
  name         = var.authorizer_name
  type         = "FRONTEND"
  function_urn = huaweicloud_fgs_function.test.urn
  identities   = ["Authorization"]
}
```

**参数说明**：
- **instance_id**：API网关实例的ID，引用前面创建的API网关实例资源的ID
- **name**：自定义认证器名称，通过引用输入变量authorizer_name进行赋值
- **type**：认证器类型，设置为"FRONTEND"
- **function_urn**：函数的URN，引用前面创建的函数资源的URN
- **identities**：认证参数列表，设置为["Authorization"]

### 11. 创建API定义资源

在TF文件中添加以下脚本以告知Terraform创建API定义资源：

```hcl
variable "api_name" {
  description = "API名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建API定义资源
resource "huaweicloud_apig_api" "test" {
  instance_id       = huaweicloud_apig_instance.test.id
  group_id         = huaweicloud_apig_group.test.id
  name             = var.api_name
  type             = "Public"
  request_protocol = "HTTPS"
  request_method   = "GET"
  request_path     = "/test"
  security_authentication = "CUSTOM"
  backend_type     = "HTTP"
  backend_path     = "/test"
  backend_method   = "GET"
  backend_address  = "https://example.com"
}
```

**参数说明**：
- **instance_id**：API网关实例的ID，引用前面创建的API网关实例资源的ID
- **group_id**：API分组的ID，引用前面创建的API网关分组资源的ID
- **name**：API名称，通过引用输入变量api_name进行赋值
- **type**：API类型，设置为"Public"
- **request_protocol**：请求协议，设置为"HTTPS"
- **request_method**：请求方法，设置为"GET"
- **request_path**：请求路径，设置为"/test"
- **security_authentication**：安全认证方式，设置为"CUSTOM"
- **backend_type**：后端类型，设置为"HTTP"
- **backend_path**：后端路径，设置为"/test"
- **backend_method**：后端方法，设置为"GET"
- **backend_address**：后端地址，设置为"https://example.com"

### 12. 预设资源部署所需的入参（可选）

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
subnet_cidr = "192.168.1.0/24"
subnet_gateway = "192.168.1.1"

# 安全组配置
secgroup_name = "tf_test_secgroup"

# API网关配置
apig_name = "tf_test_apig"
group_name = "tf_test_group"
api_name = "tf_test_api"

# 函数配置
function_name = "tf_test_function"

# 自定义认证器配置
authorizer_name = "tf_test_authorizer"
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

### 13. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建API网关自定义认证
4. 运行 `terraform show` 查看已创建的API网关自定义认证详情

## 参考信息

- [华为云API网关产品文档](https://support.huaweicloud.com/apig/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [APIG自定义认证最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/apig)
