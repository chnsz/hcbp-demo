# 使用FunctionGraph实现APIG自定义认证

## 概述

华为云API网关（APIG）支持多种认证方式，包括IAM认证、APP认证等。同时，API网关也支持用户使用自己的认证方式（以下简称自定义认证），以便更好地兼容已有业务能力。本最佳实践将介绍如何使用Terraform自动化部署API网关自定义认证，以FunctionGraph函数实现认证逻辑。

### 应用场景

- 需要使用自定义认证方式来保护API访问安全
- 需要将已有的认证系统集成到API网关中
- 需要实现复杂的认证逻辑和授权控制
- 需要对API请求进行动态的身份验证

### 方案优势

- 灵活性：可以根据业务需求自定义认证逻辑
- 可扩展性：可以轻松集成现有的认证系统
- 安全性：支持多种认证方式，确保API访问安全
- 易维护：认证逻辑集中管理，便于维护和更新

### 涉及服务

- API网关（APIG）：提供API的创建、管理、维护等功能
- 函数工作流（FunctionGraph）：提供无服务器计算平台，用于运行认证函数
- 统一身份认证服务（IAM）：提供身份认证和权限管理

## 资源/数据源设计

本最佳实践涉及以下主要资源和数据源：

### 数据源

1. **可用区（data.huaweicloud_availability_zones）**
   - 用途：获取可用区信息

### 资源

1. **VPC网络（huaweicloud_vpc）**
   - 用途：为API网关实例提供网络环境

2. **VPC子网（huaweicloud_vpc_subnet）**
   - 用途：在VPC中划分子网空间

3. **安全组（huaweicloud_networking_secgroup）**
   - 用途：控制API网关实例的网络访问

4. **安全组规则（huaweicloud_networking_secgroup_rule）**
   - 用途：配置API网关实例的访问控制规则

5. **API网关实例（huaweicloud_apig_instance）**
   - 用途：提供API网关服务

6. **API网关分组（huaweicloud_apig_group）**
   - 用途：管理和组织相关API

7. **API定义（huaweicloud_apig_api）**
   - 用途：定义需要进行自定义认证的API接口

8. **自定义认证器（huaweicloud_apig_custom_authorizer）**
   - 用途：关联FunctionGraph函数，实现认证逻辑

9. **函数（huaweicloud_fgs_function）**
   - 用途：实现自定义认证逻辑

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

## 详细配置

### 数据源配置

#### 1. 可用区（data.huaweicloud_availability_zones）

获取指定region（默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建API网关实例。

```hcl
data "huaweicloud_availability_zones" "test" {}
```

### 资源配置

#### 1. VPC网络（huaweicloud_vpc）

在指定region（默认继承当前provider块中所指定的region）下创建VPC网络，为API网关实例提供网络隔离。

```hcl
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称
- **cidr**：VPC网段，格式为CIDR

#### 2. VPC子网（huaweicloud_vpc_subnet）

在指定region（默认继承当前provider块中所指定的region）下创建子网，为API网关实例提供网络空间。

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

resource "huaweicloud_vpc_subnet" "test" {
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway
  vpc_id     = huaweicloud_vpc.test.id
}
```

**参数说明**：
- **name**：子网名称
- **cidr**：子网网段，格式为CIDR
- **gateway_ip**：网关IP地址
- **vpc_id**：VPC ID

#### 3. 安全组（huaweicloud_networking_secgroup）

在指定region（默认继承当前provider块中所指定的region）下创建安全组，控制API网关实例的网络访问。

```hcl
variable "secgroup_name" {
  description = "安全组名称"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.secgroup_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称
- **delete_default_rules**：是否删除默认规则

#### 4. 安全组规则（huaweicloud_networking_secgroup_rule）

在指定region（默认继承当前provider块中所指定的region）下配置API网关实例的访问控制规则。

```hcl
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
```

**参数说明**：
- **security_group_id**：安全组ID
- **direction**：规则方向
- **ethertype**：网络协议版本
- **protocol**：协议类型
- **port_range_min**：起始端口
- **port_range_max**：结束端口
- **remote_ip_prefix**：允许访问的IP范围
- **description**：规则描述

#### 5. API网关实例（huaweicloud_apig_instance）

在指定region（默认继承当前provider块中所指定的region）下创建API网关实例，提供API网关服务。

```hcl
variable "apig_name" {
  description = "API网关实例名称"
  type        = string
}

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
- **name**：API网关实例名称
- **edition**：实例规格
- **vpc_id**：VPC ID
- **subnet_id**：子网ID
- **security_group_id**：安全组ID
- **availability_zones**：可用区列表
- **description**：实例描述
- **enterprise_project_id**：企业项目ID

#### 6. API网关分组（huaweicloud_apig_group）

在指定region（默认继承当前provider块中所指定的region）下创建API网关分组，用于管理和组织相关API。

```hcl
variable "group_name" {
  description = "API分组名称"
  type        = string
}

resource "huaweicloud_apig_group" "test" {
  name        = var.group_name
  description = "API分组"
  instance_id = huaweicloud_apig_instance.test.id
}
```

**参数说明**：
- **name**：API分组名称
- **description**：分组描述
- **instance_id**：API网关实例ID

#### 7. API定义（huaweicloud_apig_api）

在指定region（默认继承当前provider块中所指定的region）下创建需要进行自定义认证的API定义。

```hcl
variable "api_name" {
  description = "API名称"
  type        = string
}

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
- **instance_id**：API网关实例ID
- **group_id**：API分组ID
- **name**：API名称
- **type**：API类型
- **request_protocol**：请求协议
- **request_method**：请求方法
- **request_path**：请求路径
- **security_authentication**：安全认证方式
- **backend_type**：后端类型
- **backend_path**：后端路径
- **backend_method**：后端方法
- **backend_address**：后端地址

#### 8. 自定义认证器（huaweicloud_apig_custom_authorizer）

在指定region（默认继承当前provider块中所指定的region）下创建自定义认证器，关联FunctionGraph函数。

```hcl
variable "authorizer_name" {
  description = "自定义认证器名称"
  type        = string
}

resource "huaweicloud_apig_custom_authorizer" "test" {
  instance_id  = huaweicloud_apig_instance.test.id
  name         = var.authorizer_name
  type         = "FRONTEND"
  function_urn = huaweicloud_fgs_function.test.urn
  identities   = ["Authorization"]
}
```

**参数说明**：
- **instance_id**：API网关实例ID
- **name**：自定义认证器名称
- **type**：认证器类型
- **function_urn**：函数URN
- **identities**：认证参数列表

#### 9. 函数（huaweicloud_fgs_function）

在指定region（默认继承当前provider块中所指定的region）下创建FunctionGraph函数，实现自定义认证逻辑。

```hcl
variable "function_name" {
  description = "函数名称"
  type        = string
}

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
- **name**：函数名称
- **app**：函数所属应用
- **handler**：函数入口
- **memory_size**：函数运行时内存大小
- **timeout**：函数超时时间
- **runtime**：运行时环境
- **code_type**：代码类型
- **func_code**：函数代码

### 可扩展配置

#### 1. HTTPS安全组规则（huaweicloud_networking_secgroup_rule）

在指定region（默认继承当前provider块中所指定的region）下创建允许HTTPS访问的安全组规则。

```hcl
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
- **security_group_id**：安全组ID
- **direction**：规则方向
- **ethertype**：网络协议版本
- **protocol**：协议类型
- **port_range_min**：起始端口
- **port_range_max**：结束端口
- **remote_ip_prefix**：允许访问的IP范围
- **description**：规则描述

> 该规则用于允许HTTPS（443）端口访问。建议根据实际业务需求配置remote_ip_prefix，遵循最小权限原则。

## 部署流程

1. 创建VPC和子网
2. 配置安全组和规则
3. 创建API网关实例
4. 创建API分组
5. 创建自定义认证函数
6. 配置自定义认证器
7. 创建API定义

## 操作步骤

1. **准备工作**
   - 安装Terraform
   - 配置华为云认证信息
   - 创建工作目录

2. **创建Terraform配置文件**
   ```bash
   touch main.tf
   touch variables.tf
   ```

3. **初始化和部署**
   ```bash
   terraform init
   terraform plan
   terraform apply
   ```

4. **验证部署**
   - 登录API网关控制台
   - 检查实例状态
   - 测试API访问
   - 验证认证效果

## 注意事项

1. **网络规划**
   - 合理规划VPC网段
   - 配置必要的安全组规则
   - 确保网络连通性

2. **安全配置**
   - 使用HTTPS协议
   - 合理设置认证参数
   - 实现完善的错误处理

3. **性能优化**
   - 合理配置函数内存和超时时间
   - 适当设置认证结果缓存
   - 优化认证逻辑代码

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 自动化部署的API网关自定义认证
2. 灵活的认证逻辑实现
3. 安全的API访问控制
4. 可维护的认证系统

## 参考信息

- [API网关产品文档](https://support.huaweicloud.com/apig/index.html)
- [函数工作流产品文档](https://support.huaweicloud.com/functiongraph/index.html)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs) 
- [APIG最佳实践](https://github.com/huaweicloud/terraform-provider-huaweicloud/blob/master/examples/apig/api-custom-authorizer)
