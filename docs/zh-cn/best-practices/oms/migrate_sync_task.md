# 部署迁移同步任务

## 应用场景

对象存储迁移服务（Object Storage Migration Service，OMS）是华为云提供的一站式数据迁移服务，支持将其他云服务商或本地环境中的对象数据迁移至华为云对象存储服务（OBS）。在跨云、跨区域的数据搬迁场景中，除了单次迁移之外，往往还需要持续保持源端与目的端数据的一致性。

本最佳实践将介绍如何使用Terraform自动化部署OMS迁移同步任务，包括创建源端与目的端OBS桶，以及创建迁移同步任务并配置源端云服务商、源端区域、一致性校验方式和元数据迁移等参数，帮助您快速构建可持续同步的对象数据迁移能力。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [源端OBS桶（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [目的端OBS桶（huaweicloud_obs_bucket）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/obs_bucket)
- [迁移同步任务（huaweicloud_oms_migration_sync_task）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/oms_migration_sync_task)

### 资源/数据源依赖关系

```
huaweicloud_obs_bucket.source
    └── huaweicloud_oms_migration_sync_task.test

huaweicloud_obs_bucket.dest
    └── huaweicloud_oms_migration_sync_task.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建源端OBS桶

在TF文件（如main.tf）中添加以下脚本以创建源端OBS桶：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建源端OBS桶资源
variable "source_bucket_name" {
  description = "The name of the source OBS bucket"
  type        = string
}

variable "bucket_storage_class" {
  description = "The storage class of the OBS bucket"
  type        = string
  default     = "STANDARD"
}

variable "bucket_acl" {
  description = "The ACL of the OBS bucket"
  type        = string
  default     = "private"
}

variable "bucket_force_destroy" {
  description = "Whether to force destroy the OBS bucket"
  type        = bool
  default     = true
}

resource "huaweicloud_obs_bucket" "source" {
  bucket        = var.source_bucket_name
  storage_class = var.bucket_storage_class
  acl           = var.bucket_acl
  force_destroy = var.bucket_force_destroy
}
```

**参数说明**：
- **bucket**：桶名称，通过引用输入变量 source_bucket_name 进行赋值
- **storage_class**：桶的存储类别，通过引用输入变量 bucket_storage_class 进行赋值，默认值为 STANDARD
- **acl**：桶的访问控制策略，通过引用输入变量 bucket_acl 进行赋值，默认值为 private
- **force_destroy**：是否强制删除桶，通过引用输入变量 bucket_force_destroy 进行赋值，默认值为 true

### 3. 创建目的端OBS桶

在TF文件（如main.tf）中添加以下脚本以创建目的端OBS桶：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建目的端OBS桶资源
variable "dest_bucket_name" {
  description = "The name of the destination OBS bucket"
  type        = string
}

resource "huaweicloud_obs_bucket" "dest" {
  bucket        = var.dest_bucket_name
  storage_class = var.bucket_storage_class
  acl           = var.bucket_acl
  force_destroy = var.bucket_force_destroy
}
```

**参数说明**：
- **bucket**：桶名称，通过引用输入变量 dest_bucket_name 进行赋值
- **storage_class**：桶的存储类别，通过引用输入变量 bucket_storage_class 进行赋值
- **acl**：桶的访问控制策略，通过引用输入变量 bucket_acl 进行赋值
- **force_destroy**：是否强制删除桶，通过引用输入变量 bucket_force_destroy 进行赋值

### 4. 创建迁移同步任务

在TF文件（如main.tf）中添加以下脚本以创建迁移同步任务：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建迁移同步任务资源
variable "source_cloud_type" {
  description = "The source cloud service provider"
  type        = string
  default     = "HuaweiCloud"
}

variable "source_region" {
  description = "The region where the source bucket is located"
  type        = string
}

variable "source_access_key" {
  description = "The access key for accessing the source bucket"
  type        = string
  sensitive   = true
}

variable "source_secret_key" {
  description = "The secret key for accessing the source bucket"
  type        = string
  sensitive   = true
}

variable "dest_access_key" {
  description = "The access key for accessing the destination bucket"
  type        = string
  sensitive   = true
}

variable "dest_secret_key" {
  description = "The secret key for accessing the destination bucket"
  type        = string
  sensitive   = true
}

variable "task_description" {
  description = "The description of the migration synchronization task"
  type        = string
  default     = ""
}

variable "consistency_check" {
  description = "The consistency check method"
  type        = string
  default     = "size_last_modified"
}

variable "enable_metadata_migration" {
  description = "Whether to enable metadata migration"
  type        = bool
  default     = false
}

resource "huaweicloud_oms_migration_sync_task" "test" {
  src_cloud_type            = var.source_cloud_type
  src_region                = var.source_region
  src_ak                    = var.source_access_key
  src_sk                    = var.source_secret_key
  src_bucket                = huaweicloud_obs_bucket.source.bucket
  dst_ak                    = var.dest_access_key
  dst_sk                    = var.dest_secret_key
  dst_bucket                = huaweicloud_obs_bucket.dest.bucket
  description               = var.task_description
  consistency_check         = var.consistency_check
  enable_metadata_migration = var.enable_metadata_migration
}
```

**参数说明**：
- **src_cloud_type**：源端云服务商，通过引用输入变量 source_cloud_type 进行赋值，默认值为 HuaweiCloud
- **src_region**：源端桶所在区域，通过引用输入变量 source_region 进行赋值
- **src_ak**：访问源端桶的访问密钥（AK），通过引用输入变量 source_access_key 进行赋值
- **src_sk**：访问源端桶的私有访问密钥（SK），通过引用输入变量 source_secret_key 进行赋值
- **src_bucket**：源端桶名称，通过引用源端OBS桶资源的 bucket 属性进行赋值
- **dst_ak**：访问目的端桶的访问密钥（AK），通过引用输入变量 dest_access_key 进行赋值
- **dst_sk**：访问目的端桶的私有访问密钥（SK），通过引用输入变量 dest_secret_key 进行赋值
- **dst_bucket**：目的端桶名称，通过引用目的端OBS桶资源的 bucket 属性进行赋值
- **description**：迁移同步任务描述，通过引用输入变量 task_description 进行赋值
- **consistency_check**：一致性校验方式，通过引用输入变量 consistency_check 进行赋值，默认值为 size_last_modified
- **enable_metadata_migration**：是否启用元数据迁移，通过引用输入变量 enable_metadata_migration 进行赋值，默认值为 false

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "YourAccessKey"
secret_key  = "YourSecretKey"

# 资源变量
source_bucket_name = "tf-test-source"
dest_bucket_name   = "tf-test-dest"
source_region      = "cn-north-4"
source_access_key  = "YourSourceBucketAccessKey"
source_secret_key  = "YourSourceBucketSecretKey"
dest_access_key    = "YourDestBucketAccessKey"
dest_secret_key    = "YourDestBucketSecretKey"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="source_bucket_name=my-source-bucket"`
2. 环境变量：`export TF_VAR_source_bucket_name=my-source-bucket`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建迁移同步任务
4. 运行 `terraform show` 查看已创建的迁移同步任务

## 参考信息

- [华为云OMS产品文档](https://support.huaweicloud.com/oms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [OMS迁移同步任务最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/oms/migrate-sync-task)
