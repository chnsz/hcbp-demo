# 使用FunctionGraph实现OBS文件下载

## 本最佳实践概述

在云服务场景中，经常需要从对象存储服务(OBS)中下载文件进行处理。通过函数工作流服务（FunctionGraph），您可以轻松实现OBS文件的自动下载和处理。本最佳实践将介绍如何使用函数服务实现OBS文件的下载功能。

### 应用场景

- 需要定期从OBS下载文件进行处理
- 需要对OBS中的文件进行批量下载
- 需要在文件上传到OBS后触发下载处理
- 需要基于特定事件触发文件下载

### 方案优势

- 无服务器：无需管理服务器，按需执行，降低成本
- 自动化：支持事件触发，自动执行下载任务
- 灵活性：可以根据业务需求自定义下载逻辑
- 可扩展：支持并发下载，提高处理效率

### 涉及产品

- 对象存储服务（OBS）：提供文件存储服务
- 函数工作流（FunctionGraph）：提供函数计算服务
- 统一身份认证服务（IAM）：提供身份认证和权限管理

## 资源/数据源设计

本最佳实践涉及以下主要资源：

1. **OBS桶**：
   - 用途：存储需要下载的文件
   - 功能：提供文件存储和访问能力
   - 特点：支持多种存储类型，安全可靠
   - 关键配置：桶名称、访问权限、存储类型等
   - 输入：上传的文件
   - 输出：可供下载的文件对象

2. **FunctionGraph函数**：
   - 用途：实现文件下载逻辑
   - 功能：从OBS下载文件并进行处理
   - 特点：支持Python等多种运行环境
   - 关键配置：函数代码、运行时环境、内存配置、超时时间等
   - 输入：OBS事件或定时触发
   - 输出：下载结果和处理状态

3. **IAM委托**：
   - 用途：授权函数访问OBS资源
   - 功能：提供必要的访问权限
   - 特点：细粒度的权限控制
   - 关键配置：委托名称、权限策略等
   - 安全特性：最小权限原则

### 资源依赖关系

```
OBS桶
    └── FunctionGraph函数
         └── IAM委托
```

### 脚本操作流程

1. 创建OBS桶和必要的IAM委托
2. 部署FunctionGraph函数
3. 配置触发器（OBS触发器或定时触发器）
4. 函数执行时从OBS下载文件
5. 处理下载结果并记录日志

### 注意事项

1. **性能考虑**：
   - 合理设置函数内存和超时时间
   - 考虑文件大小对下载时间的影响
   - 优化并发下载策略

2. **成本优化**：
   - 选择合适的函数规格
   - 优化触发频率
   - 合理使用资源

3. **安全性**：
   - 使用最小权限原则配置IAM权限
   - 加密敏感数据
   - 做好访问控制

### FunctionGraph函数（huaweicloud_fgs_function）

**功能概述**

创建一个FunctionGraph函数用于从OBS下载文件。

**详细配置**

```hcl
resource "huaweicloud_fgs_function" "obs_download" {
  name        = "obs-file-download"
  app         = "default"
  agency      = "function_agency"
  handler     = "index.handler"
  memory_size = 128
  timeout     = 30
  runtime     = "Python3.6"
  code_type   = "inline"
  
  func_code = <<EOF
import json
import os
import boto3
from botocore.client import Config

def handler(event, context):
    # 配置OBS客户端
    obs_client = boto3.client(
        's3',
        aws_access_key_id=os.getenv('ACCESS_KEY'),
        aws_secret_access_key=os.getenv('SECRET_KEY'),
        endpoint_url=os.getenv('OBS_ENDPOINT'),
        config=Config(signature_version='s3v4')
    )
    
    try:
        # 从事件中获取桶名和对象键
        bucket = event['Records'][0]['s3']['bucket']['name']
        key = event['Records'][0]['s3']['object']['key']
        
        # 下载文件
        download_path = f"/tmp/{key.split('/')[-1]}"
        obs_client.download_file(bucket, key, download_path)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'File downloaded successfully',
                'file': download_path
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({
                'message': 'Error downloading file',
                'error': str(e)
            })
        }
EOF
}
```

+ **name**：函数名称
+ **app**：函数所属应用
+ **agency**：委托名称，用于授权函数访问OBS
+ **handler**：函数入口，格式为 `文件名.函数名`
+ **memory_size**：函数运行时内存大小
+ **timeout**：函数超时时间
+ **runtime**：运行时环境
+ **code_type**：代码类型，这里使用内联代码
+ **func_code**：函数代码，实现OBS文件下载逻辑

### OBS触发器（huaweicloud_fgs_trigger）

**功能概述**

创建OBS触发器，当文件上传到OBS时自动触发函数执行。

**详细配置**

```hcl
resource "huaweicloud_fgs_trigger" "obs_trigger" {
  function_urn = huaweicloud_fgs_function.obs_download.urn
  type         = "OBS"
  status       = "ACTIVE"
  
  obs_trigger {
    bucket_name     = "example-bucket"
    events         = ["ObjectCreated:*"]
    prefix         = "uploads/"
    suffix         = ".txt"
  }
}
```

+ **function_urn**：函数URN
+ **type**：触发器类型
+ **status**：触发器状态
+ **bucket_name**：OBS桶名称
+ **events**：触发事件类型
+ **prefix**：对象前缀过滤器
+ **suffix**：对象后缀过滤器

### 前置准备

在部署此方案前，请确保已完成以下准备工作：

1. 已创建可用的OBS桶，用于存储需要下载的文件。
2. 已通过控制台创建IAM委托，并授予FunctionGraph访问OBS的必要权限：
   - 登录华为云控制台，进入"统一身份认证服务"
   - 创建委托，选择"云服务"作为委托类型
   - 选择"FunctionGraph"作为云服务
   - 为委托添加"OBS OperateAccess"权限
   - 记录委托名称，将在创建函数时使用

> **说明：** IAM委托是授予FunctionGraph访问其他云服务的凭证，请确保委托具有适当的OBS访问权限。

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
   - 测试OBS文件下载功能

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 一个完整的使用FunctionGraph实现OBS文件下载解决方案
2. 可复用的Terraform配置脚本
3. 灵活可定制的触发配置

## 参考信息

- [使用FunctionGraph处理OBS事件](https://support.huaweicloud.com/bestpractice-obs/obs_05_0001.html)
- [FunctionGraph开发指南](https://support.huaweicloud.com/devg-functiongraph/functiongraph_02_0101.html)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs) 