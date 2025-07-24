# 使用Terraform部署FunctionGraph函数下载OBS对象

## 概述

华为云函数工作流（FunctionGraph）是一项基于事件驱动的无服务器计算服务。本最佳实践将介绍如何使用Terraform自动化部署一个用于下载OBS对象的函数，并为其配置定时触发器。通过定时触发器，您可以按照指定的时间间隔自动执行函数来下载OBS对象，实现对象的自动化、周期性下载。

### 应用场景

- 需要自动化下载OBS对象
- 基于事件触发的对象下载
- 无服务器计算场景

### 方案优势

- 自动化部署：使用Terraform实现基础设施即代码
- 无服务器：无需管理服务器基础设施
- 按需付费：根据实际使用量计费
- 灵活触发：支持多种触发方式

### 涉及产品

- 函数工作流（FunctionGraph）：提供函数计算服务
- 对象存储服务（OBS）：提供对象存储能力
- 统一身份认证服务（IAM）：提供身份认证和权限管理

## 资源/数据源设计

本最佳实践涉及以下主要资源和数据源：

### 资源

1. **OBS桶（huaweicloud_obs_bucket）**
   - 用途：存储函数代码包

2. **OBS对象（huaweicloud_obs_bucket_object）**
   - 用途：上传函数代码包

3. **函数（huaweicloud_fgs_function）**
   - 用途：创建FunctionGraph函数

4. **触发器（huaweicloud_fgs_trigger）**
   - 用途：创建OBS事件触发器

### 资源/数据源依赖关系

```
huaweicloud_obs_bucket
    └── huaweicloud_obs_bucket_object
        └── huaweicloud_fgs_function
            └── huaweicloud_fgs_trigger
```

## 详细配置

### 资源配置

#### 1. OBS桶（huaweicloud_obs_bucket）

创建OBS桶用于存储对象。

```hcl
variable "bucket_name" {
  description = "OBS桶名称"
  type        = string
}

resource "huaweicloud_obs_bucket" "test" {
  bucket = var.bucket_name
  acl    = "private"
}
```

**参数说明**：
- **bucket**：桶名称
- **acl**：访问控制权限

#### 2. OBS对象（huaweicloud_obs_bucket_object）

上传对象到OBS桶。

```hcl
variable "object_path" {
  description = "对象在OBS桶中的路径"
  type        = string
}

variable "object_name" {
  description = "对象名称"
  type        = string
}

variable "local_file_path" {
  description = "本地文件路径"
  type        = string
}

resource "huaweicloud_obs_bucket_object" "test" {
  bucket = huaweicloud_obs_bucket.test.bucket
  key    = format("%s%s", var.object_path, var.object_name)
  source = var.local_file_path
}
```

**参数说明**：
- **bucket**：桶名称
- **key**：对象名称，由对象路径和对象名称组成
- **source**：本地文件路径

#### 3. 函数（huaweicloud_fgs_function）

创建FunctionGraph函数。

```hcl
variable "function_name" {
  description = "函数名称"
  type        = string
}

variable "agency_name" {
  description = "委托名称"
  type        = string
}

variable "obs_address" {
  description = "OBS服务地址"
  type        = string
}

resource "huaweicloud_fgs_function" "test" {
  name        = var.function_name
  app         = "default"
  agency      = var.agency_name
  description = "Download file from OBS bucket"
  handler     = "index.handler"
  depend_list = [
    "a12cea7e-8a52-4cf5-8ab5-5956d1cf7325",
  ]
  memory_size = 256
  timeout     = 15
  runtime     = "Python2.7"
  code_type   = "inline"
  user_data   = jsonencode({ "srcBucket" = var.bucket_name, "srcObjPath" = var.object_path, "srcObjName" = var.object_name, "obsAddress" = var.obs_address })
  func_code   = <<EOF
# -*- coding: utf-8 -*-
# When running this sample code to access OBS, you must specify an agency with global service access permissions (or at least with OBS access permissions).
from obs import ObsClient #Require public dependency:esdk_obs_python-3.x
import sys
import os

current_file_path = os.path.dirname(os.path.realpath(__file__))
# Adds the current path to search paths to import third-party libraries.
sys.path.append(current_file_path)

TEMP_ROOT_PATH = "/tmp/"       # Downloads a file from OBS to this directory.

# Handler of the function
def handler (event, context):
    logger = context.getLogger()                    # Obtains a log instance.

    srcBucket = context.getUserData('srcBucket')    # Enter the name of the bucket where the actual file to be downloaded is stored.
    srcObjName = context.getUserData('srcObjName')  # Enter the name of the actual file to be downloaded, for example, file.txt.
    srcObjPath = context.getUserData('srcObjPath')  # Enter the directory of the actual file to be downloaded, for example, for_download/.

    if srcBucket is None or srcObjName is None:
        logger.error("Please set environment variables srcBucket and srcObjName.")
        return ("Please set environment variables srcBucket and srcObjName.")

    # You are advised to use the log instance provided by FunctionGraph to debug or print messages and not to use the native print function.
    logger.info( "*** srcBucketName: " + srcBucket)
    logger.info("*** srcObjName:" + srcObjName)

    # Obtains a temporary AK and SK to access OBS. An agency is required to access IAM.
    ak = context.getAccessKey()
    sk = context.getSecretKey()
    if ak == "" or sk == "":
        logger.error("Failed to access OBS because no temporary AK, SK, or token has been obtained. Please set an agency.")
        return ("Failed to access OBS because no temporary AK, SK, or token has been obtained. Please set an agency.")

    obs_address = context.getUserData('obsAddress')      # Domain name of the OBS service. Use the default value.
    if obs_address is None:
        obs_address = 'obs.cn-north-1.myhuaweicloud.com'
    # Build a client.
    client = GetObsClient(obs_address, ak, sk)
    # Downloads an uploaded file from OBS.
    status = GetObject(client, srcBucket, srcObjPath, srcObjName)
    if (status == 200 or status == 201):
        image_size = cal_image_size(srcObjName)
        output = "the object you download from OBS is " + str(image_size) + " KB"
        logger.info(output)
        return output
    else:
        logger.error("download file from OBS failed")
        return ("download file from OBS failed")


# Build a obs client to get the obs object.
def GetObsClient(obsAddr, ak, sk):
    client = ObsClient(access_key_id=ak, secret_access_key=sk,server=obsAddr)
    return client


# Downloads a file from OBS to a local directory.
def GetObject(client, bucketName, objPath, objName):
    resp = client.getObject(bucketName, objPath+objName, downloadPath=TEMP_ROOT_PATH+objName)
    print('*** GetObject resp: ', resp)
    return (int(resp.status))


def cal_image_size(fileName):
    fileNamePath = TEMP_ROOT_PATH + fileName
    # Calculates the size of a file in KB.
    size = os.path.getsize(fileNamePath) / 1024
    return size

EOF
}
```

**参数说明**：
- **name**：函数名称
- **app**：函数所属应用
- **agency**：委托名称
- **description**：函数描述
- **handler**：函数入口
- **depend_list**：依赖列表
- **memory_size**：内存大小
- **timeout**：超时时间
- **runtime**：运行时
- **code_type**：代码类型
- **user_data**：用户数据，包含OBS相关配置
- **func_code**：函数代码

#### 4. 触发器（huaweicloud_fgs_trigger）

创建定时触发器。

```hcl
variable "trigger_name" {
  description = "触发器名称"
  type        = string
}

resource "huaweicloud_fgs_trigger" "test" {
  function_urn = huaweicloud_fgs_function.test.urn
  type         = "TIMER"
  status       = "ACTIVE"

  timer {
    name          = var.trigger_name
    schedule_type = "Rate"
    schedule      = "3d"
  }
}
```

**参数说明**：
- **function_urn**：函数URN
- **type**：触发器类型
- **status**：触发器状态
- **timer**：定时器配置
  - **name**：触发器名称
  - **schedule_type**：调度类型
  - **schedule**：调度周期

## 部署流程

1. 创建函数代码包
2. 创建OBS桶
3. 上传代码包到OBS
4. 创建函数

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
   - 登录华为云控制台
   - 检查函数状态
   - 测试函数执行

## 注意事项

1. **权限配置**：
   - 确保函数具有访问OBS的权限
   - 正确配置委托权限
   - 遵循最小权限原则

2. **资源规划**：
   - 合理设置函数内存
   - 设置合适的超时时间
   - 选择适当的运行时

3. **代码管理**：
   - 妥善保管函数代码
   - 定期更新运行时版本
   - 做好版本控制

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 自动化部署的函数服务
2. 可重复使用的Terraform配置
3. 基于事件驱动的OBS对象下载能力

## 参考信息

- [华为云FunctionGraph产品文档](https://support.huaweicloud.com/functiongraph/index.html)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [函数工作流最佳实践](https://github.com/huaweicloud/terraform-provider-huaweicloud/blob/master/examples/fgs/obs-download) 
