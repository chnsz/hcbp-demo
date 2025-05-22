# 使用FunctionGraph实现APIG自定义认证

## 本最佳实践概述

在API的安全认证方面，API网关提供IAM认证、APP认证等方式，帮助用户快速开放API，同时API网关也支持用户使用自己的认证方式（以下简称自定义认证），以便更好地兼容已有业务能力。

API网关支持的自定义认证需要借助函数工作流服务（FunctionGraph）实现，用户在函数工作流中创建自定义认证函数，API网关调用该函数，实现自定义认证。本最佳实践将以Basic认证为例，介绍如何使用函数服务实现自定义认证。

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

### 涉及产品

- API网关（APIG）：提供API的创建、管理、维护等功能
- 函数工作流（FunctionGraph）：提供无服务器计算平台，用于运行认证函数
- 统一身份认证服务（IAM）：提供身份认证和权限管理

## 资源/数据源设计

本最佳实践涉及以下主要资源：

1. **FunctionGraph函数**：
   - 用途：实现自定义认证逻辑
   - 功能：验证请求中的认证信息，返回认证结果
   - 特点：支持Python等多种运行环境，可自定义认证规则
   - 关键配置：函数代码、运行时环境、内存配置、超时时间等
   - 输入：API请求中的认证信息（如Authorization头）
   - 输出：认证结果（包含状态码和用户信息）

2. **API网关分组**：
   - 用途：管理和组织相关API
   - 功能：提供API的统一管理和配置
   - 特点：支持环境变量配置，便于多环境部署
   - 关键配置：分组名称、描述、环境变量等
   - 作用：作为API的逻辑分组，便于管理和维护

3. **API定义**：
   - 用途：定义需要进行自定义认证的API接口
   - 功能：配置API的请求方法、路径、认证方式等
   - 特点：支持多种后端类型，可灵活配置认证方式
   - 关键配置：请求协议、方法、路径、后端服务、认证方式等
   - 安全特性：支持HTTPS协议，可配置自定义认证器

4. **自定义认证器**：
   - 用途：关联FunctionGraph函数，实现认证逻辑
   - 功能：处理API请求的认证过程
   - 特点：支持缓存配置，提高认证效率
   - 关键配置：函数关联、缓存时间、身份来源等
   - 扩展性：可自定义认证参数和处理逻辑

### 资源依赖关系

```
API网关分组
    └── API定义
         └── 自定义认证器
              └── FunctionGraph函数
```

### 认证流程

1. 客户端发起API请求，携带认证信息
2. API网关接收请求，识别需要自定义认证
3. 调用自定义认证器关联的FunctionGraph函数
4. 函数验证认证信息，返回认证结果
5. API网关根据认证结果决定是否允许访问

### 注意事项

1. **安全性考虑**：
   - 使用HTTPS协议保护API通信
   - 合理设置认证信息的缓存时间
   - 实现完善的错误处理机制

2. **性能优化**：
   - 合理配置函数内存和超时时间
   - 适当设置认证结果缓存
   - 优化认证逻辑代码

3. **可维护性**：
   - 使用环境变量管理配置
   - 统一API分组管理
   - 做好认证日志记录

### FunctionGraph函数（huaweicloud_fgs_function）

**功能概述**

创建一个FunctionGraph函数作为自定义认证器，用于验证API请求中的认证信息。

**详细配置**

```hcl
resource "huaweicloud_fgs_function" "authorizer" {
  name        = "api-custom-authorizer"
  app         = "default"
  agency      = "function_agency"
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

+ **name**：函数名称
+ **app**：函数所属应用
+ **agency**：委托名称，用于授权函数访问其他云服务
+ **handler**：函数入口，格式为 `文件名.函数名`
+ **memory_size**：函数运行时内存大小
+ **timeout**：函数超时时间
+ **runtime**：运行时环境
+ **code_type**：代码类型，这里使用内联代码
+ **func_code**：函数代码，实现认证逻辑

### API网关分组（huaweicloud_apig_group）

**功能概述**

创建API网关分组，用于管理和组织相关API。

**详细配置**

```hcl
resource "huaweicloud_apig_group" "test" {
  name        = "apig-group-basic"
  description = "基础认证API分组"
  
  # 环境变量配置
  environment = {
    STAGE = "RELEASE"
  }
}
```

+ **name**：API分组名称
+ **description**：API分组描述
+ **environment**：环境变量配置

### API定义（huaweicloud_apig_api）

**功能概述**

创建需要进行自定义认证的API定义。

**详细配置**

```hcl
resource "huaweicloud_apig_api" "api_with_auth" {
  group_id    = huaweicloud_apig_group.test.id
  name        = "api-with-custom-auth"
  type        = 1
  description = "使用自定义认证的API"
  
  request_protocol = "HTTPS"
  request_method   = "GET"
  request_path     = "/protected/resource"
  
  backend_type  = "FUNCTION"
  backend_path  = "/"
  
  security_authentication = "CUSTOM"
  custom_authorizer_id    = huaweicloud_apig_custom_authorizer.basic_auth.id
  
  web_backend_parameters {
    name     = "Authorization"
    location = "HEADER"
    type     = "STRING"
    required = true
  }
  
  response_type = "JSON"
}
```

+ **group_id**：API所属分组ID
+ **name**：API名称
+ **type**：API类型，1表示公开API
+ **request_protocol**：请求协议
+ **request_method**：请求方法
+ **request_path**：请求路径
+ **backend_type**：后端类型
+ **backend_path**：后端路径
+ **security_authentication**：安全认证方式
+ **custom_authorizer_id**：自定义认证器ID

### 自定义认证器（huaweicloud_apig_custom_authorizer）

**功能概述**

创建自定义认证器，关联FunctionGraph函数。

**详细配置**

```hcl
resource "huaweicloud_apig_custom_authorizer" "basic_auth" {
  name            = "basic-auth"
  function_urn    = huaweicloud_fgs_function.authorizer.urn
  type            = "FRONTEND"
  cache_age       = 60
  identity_source = "$request.header.Authorization"
  
  # 认证器参数配置
  parameters {
    name     = "Authorization"
    location = "HEADER"
    value    = "$request.header.Authorization"
  }
}
```

+ **name**：认证器名称
+ **function_urn**：关联的函数URN
+ **type**：认证器类型，FRONTEND表示前端认证
+ **cache_age**：缓存时间
+ **identity_source**：身份来源
+ **parameters**：认证器参数配置

## 操作步骤

1. **准备工作**
   - 确保已安装Terraform
   - 配置华为云认证信息
   - 创建工作目录并初始化Terraform

2. **创建Terraform配置文件**
   - 创建main.tf文件
   - 配置provider信息
   - 添加资源定义

3. **部署资源**
   ```bash
   terraform init
   terraform apply
   ```

4. **验证部署**
   - 在华为云控制台查看创建的资源
   - 测试API的认证功能

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 一个完整的APIG自定义认证解决方案
2. 可复用的Terraform配置脚本
3. 灵活可定制的认证逻辑
4. 安全可靠的API访问控制

## 参考信息

- [API网关自定义认证说明](https://support.huaweicloud.com/usermanual-apig/apig-ug-180621090.html)
- [FunctionGraph开发指南](https://support.huaweicloud.com/devg-functiongraph/functiongraph_02_0101.html)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs) 