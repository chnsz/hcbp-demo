# 部署图元数据

## 应用场景

图引擎服务（Graph Engine Service，GES）的图元数据用于定义图数据的标签（Label）及其属性（Property），是构建图模型、进行点边数据导入与分析的基础。通过元数据，用户可以规范图中顶点和边的数据结构，确保后续数据导入与查询的一致性。

本最佳实践将介绍如何使用Terraform自动化部署GES图元数据，包括创建用于存放元数据Schema文件的OBS桶，以及基于该桶创建GES元数据。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [对象存储桶（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [图引擎服务元数据（huaweicloud_ges_metadata）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ges_metadata)

### 资源/数据源依赖关系

```
huaweicloud_obs_bucket
    └── huaweicloud_ges_metadata
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建OBS桶

在TF文件（如main.tf）中添加以下脚本以创建用于存放GES元数据Schema文件的OBS桶：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建OBS桶资源
variable "bucket_name" {
  description = "The OBS bucket name for storing GES metadata schema files"
  type        = string
}

resource "huaweicloud_obs_bucket" "test" {
  bucket        = var.bucket_name
  acl           = "private"
  force_destroy = true
}
```

**参数说明**：

- **bucket**：桶名称，通过引用输入变量 bucket_name 进行赋值，该名称需在所有华为云账号中全局唯一
- **acl**：桶的访问权限，设置为 `private`，表示仅桶拥有者可以访问
- **force_destroy**：是否强制删除桶，设置为 `true`，表示删除桶时会同时删除桶内所有对象

### 3. 创建GES元数据

在TF文件（如main.tf）中添加以下脚本以创建GES元数据：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GES元数据资源
variable "metadata_name" {
  description = "The GES metadata name"
  type        = string
}

variable "metadata_description" {
  description = "The description of the GES metadata"
  type        = string
}

variable "metadata_schema_file" {
  description = "The schema file name in the OBS bucket"
  type        = string
}

variable "metadata_properties" {
  description = "The properties of the GES metadata label"
  type        = list(any)
  nullable    = false
}

resource "huaweicloud_ges_metadata" "test" {
  name          = var.metadata_name
  description   = var.metadata_description
  metadata_path = "${huaweicloud_obs_bucket.test.bucket}/${var.metadata_schema_file}"

  ges_metadata {
    labels {
      name       = "user"
      properties = var.metadata_properties
    }
  }

  depends_on = [
    huaweicloud_obs_bucket.test
  ]
}
```

**参数说明**：

- **name**：元数据名称，通过引用输入变量 metadata_name 进行赋值
- **description**：元数据描述，通过引用输入变量 metadata_description 进行赋值
- **metadata_path**：元数据Schema文件在OBS桶中的路径，由OBS桶名称与输入变量 metadata_schema_file 拼接而成
- **ges_metadata**：元数据定义块，其中 `labels` 用于定义标签，`name` 为标签名称，`properties` 为标签属性列表，通过引用输入变量 metadata_properties 进行赋值
- **depends_on**：显式声明对OBS桶资源的依赖，确保元数据在桶创建完成后再创建

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# OBS桶配置
bucket_name = "tf-test-ges-metadata-bucket"

# GES元数据配置
metadata_name        = "tf_test_ges_metadata"
metadata_description = "This is a demo"
metadata_schema_file = "schema_demo.xml"
metadata_properties  = [{
  dataType    = "char"
  name        = "sex"
  cardinality = "single"
}]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="bucket_name=my-bucket"`
2. 环境变量：`export TF_VAR_bucket_name=my-bucket`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建GES元数据
4. 运行 `terraform show` 查看已创建的GES元数据

## 参考信息

- [华为云图引擎服务产品文档](https://support.huaweicloud.com/ges/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GES图元数据最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ges/metadata)
