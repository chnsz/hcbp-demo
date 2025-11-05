# 部署EG触发器

## 应用场景

函数工作流（FunctionGraph）的EG触发器（EventGrid Trigger）是一种基于事件网格服务的触发器类型，可以监控和响应华为云资源的操作事件。通过EG触发器，您可以实现事件驱动的图像处理、文件转换、数据同步、实时响应等功能。

EG触发器特别适用于需要实时监控OBS对象存储事件、进行图像处理、实现自动化工作流等场景，如图像缩略图生成、文件格式转换、数据备份、事件通知等。本最佳实践将介绍如何使用Terraform自动化部署一个带有EG触发器的FunctionGraph函数，实现OBS文件上传时自动生成缩略图的功能。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [FunctionGraph依赖包查询数据源（data.huaweicloud_fgs_dependencies）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/fgs_dependencies)
- [FunctionGraph依赖包版本查询数据源（data.huaweicloud_fgs_dependency_versions）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/fgs_dependency_versions)
- [事件网格事件通道查询数据源（data.huaweicloud_eg_event_channels）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/eg_event_channels)

### 资源

- [OBS桶资源（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [FunctionGraph函数资源（huaweicloud_fgs_function）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/fgs_function)
- [FunctionGraph函数触发器资源（huaweicloud_fgs_function_trigger）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/fgs_function_trigger)

### 资源/数据源依赖关系

```
data.huaweicloud_fgs_dependencies
    └── data.huaweicloud_fgs_dependency_versions
            └── huaweicloud_fgs_function

data.huaweicloud_eg_event_channels
    └── huaweicloud_fgs_function_trigger

huaweicloud_obs_bucket
    └── huaweicloud_fgs_function_trigger

huaweicloud_fgs_function
    └── huaweicloud_fgs_function_trigger
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../docs/introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建OBS存储桶

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建OBS存储桶资源：

```hcl
# Variable definitions for OBS buckets
variable "source_bucket_name" {
  description = "The source bucket name of the OBS service"
  type        = string
}

variable "target_bucket_name" {
  description = "The target bucket name of the OBS service"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建源OBS存储桶资源
resource "huaweicloud_obs_bucket" "source" {
  bucket        = var.source_bucket_name
  acl           = "private"
  force_destroy = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建目标OBS存储桶资源
resource "huaweicloud_obs_bucket" "target" {
  bucket        = var.target_bucket_name
  acl           = "private"
  force_destroy = true
}
```

**参数说明**：
- **bucket**：OBS存储桶的名称，通过引用输入变量source_bucket_name和target_bucket_name进行赋值
- **acl**：存储桶的访问控制列表，设置为"private"表示私有访问
- **force_destroy**：是否强制删除存储桶，设置为true表示允许删除非空存储桶

### 3. 通过数据源查询FunctionGraph依赖包信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行数据源查询，其查询结果用于创建FunctionGraph函数：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的FunctionGraph公共依赖包信息，用于创建FunctionGraph函数资源
data "huaweicloud_fgs_dependencies" "test" {
  type = "public"
  name = "pillow-7.1.2"
}

# 获取指定依赖包的版本信息，用于创建FunctionGraph函数资源
data "huaweicloud_fgs_dependency_versions" "test" {
  dependency_id = try(data.huaweicloud_fgs_dependencies.test.packages[0].id, "NOT_FOUND")
  version       = 1
}
```

**参数说明**：
- **type**：依赖包类型，设置为"public"表示查询公共依赖包
- **name**：依赖包名称，设置为"pillow-7.1.2"表示查询Pillow图像处理库
- **dependency_id**：依赖包ID，通过引用数据源huaweicloud_fgs_dependencies的返回结果进行赋值
- **version**：依赖包版本，设置为1表示查询第一个版本

### 4. 通过数据源查询事件网格事件通道信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行数据源查询，其查询结果用于创建FunctionGraph EG触发器：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的事件网格事件通道信息，用于创建FunctionGraph EG触发器资源
data "huaweicloud_eg_event_channels" "test" {
  provider_type = "OFFICIAL"
  name          = "default"
}
```

**参数说明**：
- **provider_type**：事件通道提供者类型，设置为"OFFICIAL"表示查询官方事件通道
- **name**：事件通道名称，设置为"default"表示查询默认事件通道

### 5. 创建FunctionGraph函数

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建FunctionGraph函数资源：

```hcl
# Variable definitions for FunctionGraph resources
variable "function_name" {
  description = "The name of the FunctionGraph function"
  type        = string
}

variable "function_agency_name" {
  description = "The agency name that the FunctionGraph function uses"
  type        = string
}

variable "function_memory_size" {
  description = "The memory size of the function in MB"
  type        = number
  default     = 256
}

variable "function_timeout" {
  description = "The timeout of the function in seconds"
  type        = number
  default     = 40
}

variable "function_runtime" {
  description = "The runtime of the function"
  type        = string
  default     = "Python3.6"
}

variable "function_code" {
  description = "The source code of the function"
  type        = string
  default     = <<EOT
# -*-coding:utf-8 -*-
import os
import string
import random
import urllib.parse
import shutil
import contextlib
from PIL import Image
from obs import ObsClient

LOCAL_MOUNT_PATH = '/tmp/'

def handler(event, context):
    ak = context.getSecurityAccessKey()
    sk = context.getSecuritySecretKey()
    st = context.getSecurityToken()
    if ak == "" or sk == "" or st == "":
        context.getLogger().error('Failed to access OBS because no temporary '
                                  'AK, SK, or token has been obtained. Please '
                                  'set an agency.')
        return 'Failed to access OBS because no temporary AK, SK, or token ' \
               'has been obtained. Please set an agency. '

    obs_endpoint = context.getUserData('obs_endpoint')
    if not obs_endpoint:
        return 'obs_endpoint is not configured'

    output_bucket = context.getUserData('output_bucket')
    if not output_bucket:
        return 'output_bucket is not configured'

    compress_handler = ThumbnailHandler(context)
    with contextlib.ExitStack() as stack:
        # After upload the thumbnail image to obs, remove the local image
        stack.callback(shutil.rmtree,compress_handler.download_dir)
        data = event.get("data", None)
        return compress_handler.run(data)

class ThumbnailHandler:
    def __init__(self, context):
        self.logger = context.getLogger()
        obs_endpoint = context.getUserData("obs_endpoint")
        self.obs_client = new_obs_client(context, obs_endpoint)
        self.download_dir = gen_local_download_path()
        self.image_local_path = None
        self.src_bucket = None
        self.src_object_key = None
        self.output_bucket = context.getUserData("output_bucket")

    def parse_record(self, record):
        # parses the record to get src_bucket and input_object_key
        (src_bucket, src_object_key) = get_obs_obj_info(record)
        src_object_key = urllib.parse.unquote_plus(src_object_key)
        self.logger.info("src bucket name: %s", src_bucket)
        self.logger.info("src object key: %s", src_object_key)
        self.src_bucket = src_bucket
        self.src_object_key = src_object_key
        self.image_local_path = self.download_dir + src_object_key

    def run(self, record):
        self.parse_record(record)
        # Download the original image from obs to local tmp dir.
        if not self.download_from_obs():
            return "ERROR"
        # Thumbnail original image
        thumbnail_object_key, thumbnail_file_path = self.thumbnail()
        self.logger.info("thumbnail_object_key: %s, thumbnail_file_path: %s",
                        thumbnail_object_key, thumbnail_file_path)
        # Upload thumbnail image to obs
        if not self.upload_file_to_obs(thumbnail_object_key,
                                    thumbnail_file_path):
            return "ERROR"
        return "OK"

    def download_from_obs(self):
        resp = self.obs_client. \
            getObject(self.src_bucket, self.src_object_key,
                      downloadPath=self.image_local_path)
        if resp.status < 300:
            return True
        else:
            self.logger.error('failed to download from obs, '
                              'errorCode: %s, errorMessage: %s' % (
                                  resp.errorCode, resp.errorMessage))
            return False

    def upload_file_to_obs(self, object_key, local_file):
        resp = self.obs_client.putFile(self.output_bucket,
                                       object_key, local_file)
        if resp.status < 300:
            return True
        else:
            self.logger.error('failed to upload file to obs, errorCode: %s, '
                              'errorMessage: %s' % (resp.errorCode,
                                                    resp.errorMessage))
            return False

    def thumbnail(self):
        image = Image.open(self.image_local_path)
        image_w, image_h = image.size
        new_image_w, new_image_h = (int(image_w / 2), int(image_h / 2))

        image.thumbnail((new_image_w, new_image_h), resample=Image.LANCZOS)

        (path, filename) = os.path.split(self.src_object_key)
        if path != "" and not path.endswith("/"):
            path = path + "/"
        (filename, ext) = os.path.splitext(filename)
        ext_lower = ext.lower()

        thumbnail_object_key = path + filename + \
                           "_thumbnail_h_" + str(new_image_h) + \
                           "_w_" + str(new_image_w) + ext

        if hasattr(image, '_getexif'):
            image.info.pop('exif', None)

        if new_image_w * new_image_h < 500000:
            webp_ext = '.webp'
            thumbnail_object_key = path + filename + \
                                  "_thumbnail_h_" + str(new_image_h) + \
                                  "_w_" + str(new_image_w) + webp_ext
            thumbnail_file_path = self.download_dir + path + filename + \
                                 "_thumbnail_h_" + str(new_image_h) + \
                                 "_w_" + str(new_image_w) + webp_ext
            image.save(thumbnail_file_path, format='WEBP', quality=80, method=6)
        elif ext_lower in ['.jpg', '.jpeg']:
            thumbnail_file_path = self.download_dir + path + filename + \
                                 "_thumbnail_h_" + str(new_image_h) + \
                                 "_w_" + str(new_image_w) + ext
            image.save(thumbnail_file_path, quality=85, optimize=True, progressive=True)
        elif ext_lower == '.png':
            thumbnail_file_path = self.download_dir + path + filename + \
                                 "_thumbnail_h_" + str(new_image_h) + \
                                 "_w_" + str(new_image_w) + ext
            image.save(thumbnail_file_path, optimize=True, compress_level=6)
        else:
            thumbnail_file_path = self.download_dir + path + filename + \
                                 "_thumbnail_h_" + str(new_image_h) + \
                                 "_w_" + str(new_image_w) + ext
            image.save(thumbnail_file_path, quality=85, optimize=True)

        return thumbnail_object_key, thumbnail_file_path

# generate a temporary directory for downloading things
# from OBS and compress them.
def gen_local_download_path():
    letters = string.ascii_letters
    download_dir = LOCAL_MOUNT_PATH + ''.join(
        random.choice(letters) for i in range(16)) + '/'
    os.makedirs(download_dir)
    return download_dir

def new_obs_client(context, obs_server):
    ak = context.getSecurityAccessKey()
    sk = context.getSecuritySecretKey()
    st = context.getSecurityToken()
    return ObsClient(access_key_id=ak, secret_access_key=sk, security_token=st, server=obs_server)

def get_obs_obj_info(record):
    if 's3' in record:
        s3 = record['s3']
        return s3['bucket']['name'], s3['object']['key']
    else:
        obs_info = record['obs']
        return obs_info['bucket']['name'], obs_info['object']['key']
EOT
}

variable "function_description" {
  description = "The description of the function"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建FunctionGraph函数资源
resource "huaweicloud_fgs_function" "test" {
  name        = var.function_name
  agency      = var.function_agency_name
  app         = "default"
  handler     = "index.handler"
  memory_size = var.function_memory_size
  timeout     = var.function_timeout
  runtime     = var.function_runtime
  code_type   = "inline"
  func_code   = base64encode(var.function_code)
  description = var.function_description
  depend_list = data.huaweicloud_fgs_dependency_versions.test.versions[*].id

  user_data = jsonencode({
    "output_bucket" = var.target_bucket_name
    "obs_endpoint"  = format("obs.%s.myhuaweicloud.com", var.region_name)
  })
}
```

**参数说明**：
- **name**：FunctionGraph函数的名称，通过引用输入变量function_name进行赋值
- **agency**：函数的委托名称，通过引用输入变量function_agency_name进行赋值，用于函数访问其他华为云服务的权限
- **app**：函数所属的应用名称，设置为"default"表示使用默认应用
- **handler**：函数的入口点，设置为"index.handler"表示处理函数在index.py文件中的handler方法
- **memory_size**：函数的内存大小（MB），通过引用输入变量function_memory_size进行赋值，默认值为256MB
- **timeout**：函数的超时时间（秒），通过引用输入变量function_timeout进行赋值，默认值为40秒
- **runtime**：函数的运行时环境，通过引用输入变量function_runtime进行赋值，默认值为Python3.6
- **code_type**：代码类型，设置为"inline"表示内联代码
- **func_code**：函数的源代码（用于自动压缩图片），通过base64编码输入变量function_code进行赋值
- **description**：函数的描述信息，通过引用输入变量function_description进行赋值
- **depend_list**：函数依赖包列表，通过引用数据源huaweicloud_fgs_dependency_versions的返回结果进行赋值
- **user_data**：用户自定义数据，以JSON格式包含输出桶名称和OBS端点信息

### 6. 创建FunctionGraph EG触发器

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建FunctionGraph EG触发器资源：

```hcl
# Variable definitions for EG trigger
variable "trigger_status" {
  description = "The status of the timer trigger"
  type        = string
  default     = "ACTIVE"
}

variable "trigger_name_suffix" {
  description = "The suffix name of the OBS trigger"
  type        = string
}

variable "trigger_agency_name" {
  description = "The agency name that the OBS trigger uses"
  type        = string
  default     = "fgs_to_eg"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建FunctionGraph EG触发器资源
resource "huaweicloud_fgs_function_trigger" "test" {
  depends_on = [huaweicloud_obs_bucket.source]

  function_urn                   = huaweicloud_fgs_function.test.urn
  type                           = "EVENTGRID"
  cascade_delete_eg_subscription = true
  status                         = var.trigger_status
  event_data                     = jsonencode({
    "channel_id"           = try(data.huaweicloud_eg_event_channels.test.channels[0].id, "")
    "channel_name"         = try(data.huaweicloud_eg_event_channels.test.channels[0].name, "")
    "source_name"          = "HC.OBS.DWR"
    "trigger_name"         = var.trigger_name_suffix
    "agency"               = var.trigger_agency_name
    "bucket"               = var.source_bucket_name
    "event_types"          = ["OBS:DWR:ObjectCreated:PUT", "OBS:DWR:ObjectCreated:POST"]
    "Key_encode"           = true
  })

  lifecycle {
    ignore_changes = [
      event_data,
    ]
  }
}
```

**参数说明**：
- **depends_on**：显式依赖关系，确保源OBS桶在触发器创建前已存在
- **function_urn**：触发器关联的FunctionGraph函数的URN，通过引用huaweicloud_fgs_function.test.urn进行赋值
- **type**：触发器类型，设置为"EVENTGRID"表示EG触发器
- **cascade_delete_eg_subscription**：是否级联删除EG订阅，设置为true表示删除函数时同时删除EG订阅
- **status**：触发器的状态，通过引用输入变量trigger_status进行赋值，默认值为"ACTIVE"表示激活状态
- **event_data**：触发器的事件数据，以JSON格式包含以下参数：
  - **channel_id**：事件通道ID，通过引用数据源huaweicloud_eg_event_channels的返回结果进行赋值
  - **channel_name**：事件通道名称，通过引用数据源huaweicloud_eg_event_channels的返回结果进行赋值
  - **source_name**：事件源名称，设置为"HC.OBS.DWR"表示OBS数据工作流事件源
  - **trigger_name**：触发器名称后缀，通过引用输入变量trigger_name_suffix进行赋值
  - **agency**：触发器使用的委托名称，通过引用输入变量trigger_agency_name进行赋值
  - **bucket**：监控的OBS桶名称，通过引用输入变量source_bucket_name进行赋值
  - **event_types**：监控的事件类型，设置为监控OBS对象创建事件
  - **Key_encode**：是否对对象键进行编码，设置为true表示启用编码

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# OBS存储桶配置
source_bucket_name   = "tf-test-bucket-source"
target_bucket_name   = "tf-test-bucket-target"

# 函数基本信息
function_name        = "tf-test-function-image-thumbnail"
function_agency_name = "function_all_trust"

# 触发器配置
trigger_name_suffix  = "demo"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="function_name=my-function" -var="source_bucket_name=my-bucket"`
2. 环境变量：`export TF_VAR_function_name=my-function`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建OBS存储桶、FunctionGraph函数和EG触发器
4. 运行 `terraform show` 查看已创建的OBS存储桶、FunctionGraph函数和EG触发器

## 参考信息

- [华为云FunctionGraph产品文档](https://support.huaweicloud.com/functiongraph/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [FunctionGraph EG触发器最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/fgs/triggers/eg)
