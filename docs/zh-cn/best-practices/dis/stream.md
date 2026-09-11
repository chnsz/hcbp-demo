# 部署DIS通道

## 应用场景

数据接入服务（Data Ingestion Service，DIS）是华为云提供的实时数据接入服务，用于将海量数据实时采集并传输至云上，支持多种数据源与数据类型的接入。DIS通道是数据接入的基本单元，负责承载数据的写入与读取，并可通过分区数量、数据保留周期等参数满足不同吞吐量与可靠性要求。

本最佳实践将介绍如何使用Terraform自动化部署一个DIS通道，包括通道基础配置、自动扩缩容分区配置、数据格式与压缩配置以及标签管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DIS通道（huaweicloud_dis_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dis_stream)

### 资源/数据源依赖关系

```
huaweicloud_dis_stream
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DIS通道

在TF文件（如main.tf）中添加以下脚本以创建DIS通道：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DIS通道资源
variable "stream_name" {
  description = "The name of the DIS stream"
  type        = string
}

variable "stream_partition_count" {
  description = "The number of partitions for the DIS stream"
  type        = number
}

variable "stream_type" {
  description = "The type of the DIS stream"
  type        = string
  default     = null
}

variable "stream_retention_period" {
  description = "The data retention period in hours"
  type        = number
  default     = 24
}

variable "stream_auto_scale_min_partition_count" {
  description = "The minimum number of partitions for auto scaling"
  type        = number
  default     = null
}

variable "stream_auto_scale_max_partition_count" {
  description = "The maximum number of partitions for auto scaling"
  type        = number
  default     = null
}

variable "stream_compression_format" {
  description = "The compression format of the data"
  type        = string
  default     = null
}

variable "stream_data_type" {
  description = "The type of the data"
  type        = string
  default     = null
}

variable "stream_csv_delimiter" {
  description = "The delimiter for CSV data"
  type        = string
  default     = null
}

variable "stream_data_schema" {
  description = "The schema of the data"
  type        = string
  default     = null
}

variable "stream_tags" {
  description = "The key/value pairs to associate with the DIS stream"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_dis_stream" "test" {
  stream_name      = var.stream_name
  partition_count  = var.stream_partition_count
  stream_type      = var.stream_type
  retention_period = var.stream_retention_period

  auto_scale_min_partition_count = var.stream_auto_scale_min_partition_count
  auto_scale_max_partition_count = var.stream_auto_scale_max_partition_count

  compression_format = var.stream_compression_format
  data_type          = var.stream_data_type
  csv_delimiter      = var.stream_csv_delimiter
  data_schema        = var.stream_data_schema

  tags = var.stream_tags
}
```

**参数说明**：
- **stream_name**：通过引用输入变量 stream_name 进行赋值，用于指定DIS通道的名称
- **partition_count**：通过引用输入变量 stream_partition_count 进行赋值，用于指定DIS通道的分区数量
- **stream_type**：通过引用输入变量 stream_type 进行赋值，用于指定DIS通道的类型，可选值为**COMMON**（普通通道）和**ADVANCED**（高级通道）
- **retention_period**：通过引用输入变量 stream_retention_period 进行赋值，用于指定数据保留时长（单位：小时），取值范围为24至72
- **auto_scale_min_partition_count**：通过引用输入变量 stream_auto_scale_min_partition_count 进行赋值，用于指定自动扩缩容的最小分区数量
- **auto_scale_max_partition_count**：通过引用输入变量 stream_auto_scale_max_partition_count 进行赋值，用于指定自动扩缩容的最大分区数量
- **compression_format**：通过引用输入变量 stream_compression_format 进行赋值，用于指定数据的压缩格式，可选值为**zip**、**gzip**、**snappy**、**lz4**和**zstd**
- **data_type**：通过引用输入变量 stream_data_type 进行赋值，用于指定数据的类型，可选值为**CSV**、**JSON**和**BLOB**
- **csv_delimiter**：通过引用输入变量 stream_csv_delimiter 进行赋值，用于指定CSV数据的分隔符
- **data_schema**：通过引用输入变量 stream_data_schema 进行赋值，用于指定JSON格式的数据Schema
- **tags**：通过引用输入变量 stream_tags 进行赋值，用于指定DIS通道的标签键值对

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# DIS通道配置
stream_name                           = "tf_test_dis_stream"
stream_partition_count                = 2
stream_auto_scale_min_partition_count = 2
stream_auto_scale_max_partition_count = 4
stream_type                           = "COMMON"
stream_compression_format             = "zip"
stream_data_type                      = "CSV"
stream_csv_delimiter                  = ";"
stream_data_schema                    = "{\"type\":\"record\",\"name\":\"RecordName\",\"fields\":[{\"type\":\"string\",\"name\":\"name\"}]}"
stream_tags = {
  foo = "bar"
  key = "value"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="stream_name=my-stream"`
2. 环境变量：`export TF_VAR_stream_name=my-stream`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DIS通道
4. 运行 `terraform show` 查看已创建的DIS通道

## 参考信息

- [华为云数据接入服务产品文档](https://support.huaweicloud.com/dis/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DIS通道最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dis/stream)
