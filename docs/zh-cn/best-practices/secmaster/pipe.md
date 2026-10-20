# 部署数据管道

## 应用场景

安全云脑（SecMaster）是华为云原生的新一代安全运营中心，提供云上资产管理、安全态势管理、安全信息和事件管理、安全编排与自动响应等能力。数据管道用于在数据空间内对安全数据进行采集、分区存储与索引映射，是安全数据分析与检索的基础。

本最佳实践将介绍如何使用Terraform自动化部署安全云脑数据管道，包括工作空间创建、数据空间创建以及数据管道的分区、存储周期和索引映射配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [安全云脑工作空间（huaweicloud_secmaster_workspace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_workspace)
- [安全云脑数据空间（huaweicloud_secmaster_dataspace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_dataspace)
- [安全云脑数据管道（huaweicloud_secmaster_pipe）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_pipe)

### 资源/数据源依赖关系

```
huaweicloud_secmaster_workspace
    └── huaweicloud_secmaster_dataspace
            └── huaweicloud_secmaster_pipe
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建安全云脑工作空间

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全云脑工作空间资源
variable "workspace_name" {
  description = "安全云脑工作空间的名称"
  type        = string
}

variable "workspace_description" {
  description = "安全云脑工作空间的描述"
  type        = string
  default     = "Created by Terraform"
}

resource "huaweicloud_secmaster_workspace" "test" {
  name         = var.workspace_name
  project_name = var.region_name
  description  = var.workspace_description
}
```

**参数说明**：
- **name**：工作空间名称，通过引用输入变量 workspace_name 进行赋值
- **project_name**：工作空间所属项目名称，通过引用输入变量 region_name 进行赋值
- **description**：工作空间描述，通过引用输入变量 workspace_description 进行赋值

### 3. 创建安全云脑数据空间

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全云脑数据空间资源
variable "dataspace_name" {
  description = "安全云脑数据空间的名称"
  type        = string
}

variable "dataspace_description" {
  description = "安全云脑数据空间的描述"
  type        = string
  default     = "Created by Terraform"
}

resource "huaweicloud_secmaster_dataspace" "test" {
  workspace_id   = huaweicloud_secmaster_workspace.test.id
  dataspace_name = var.dataspace_name
  description    = var.dataspace_description
}
```

**参数说明**：
- **workspace_id**：数据空间所属工作空间ID，引用上一步创建工作空间的ID进行赋值
- **dataspace_name**：数据空间名称，通过引用输入变量 dataspace_name 进行赋值
- **description**：数据空间描述，通过引用输入变量 dataspace_description 进行赋值

### 4. 创建安全云脑数据管道

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全云脑数据管道资源
variable "pipe_name" {
  description = "数据管道的名称"
  type        = string
}

variable "shards" {
  description = "数据管道的分区数量，取值范围为1-64"
  type        = number
  default     = 3
}

variable "storage_period" {
  description = "数据保留周期（天），取值范围为7-180"
  type        = number
  default     = 30
}

variable "pipe_description" {
  description = "数据管道的描述"
  type        = string
  default     = "Created by Terraform"
}

variable "timestamp_field" {
  description = "数据管道的时间戳字段"
  type        = string
  default     = "timestamp"
}

variable "pipe_status" {
  description = "数据管道的状态，有效值为open、closed"
  type        = string
  default     = "open"
}

resource "huaweicloud_secmaster_pipe" "test" {
  workspace_id    = huaweicloud_secmaster_workspace.test.id
  dataspace_id    = huaweicloud_secmaster_dataspace.test.id
  pipe_name       = var.pipe_name
  shards          = var.shards
  storage_period  = var.storage_period
  description     = var.pipe_description
  timestamp_field = var.timestamp_field
  status          = var.pipe_status

  mapping = jsonencode({
    id   = {
      is_chinese_exist = true
      properties       = {}
      type             = "text"
    }
    name = {
      is_chinese_exist = false
      properties       = {}
      type             = "text"
    }
  })
}
```

**参数说明**：
- **workspace_id**：数据管道所属工作空间ID，引用工作空间资源的ID进行赋值
- **dataspace_id**：数据管道所属数据空间ID，引用数据空间资源的ID进行赋值
- **pipe_name**：数据管道名称，通过引用输入变量 pipe_name 进行赋值，创建后不可修改
- **shards**：数据管道分区数量，通过引用输入变量 shards 进行赋值
- **storage_period**：数据保留周期（天），通过引用输入变量 storage_period 进行赋值
- **description**：数据管道描述，通过引用输入变量 pipe_description 进行赋值
- **timestamp_field**：数据管道时间戳字段，通过引用输入变量 timestamp_field 进行赋值
- **status**：数据管道状态，通过引用输入变量 pipe_status 进行赋值
- **mapping**：索引映射配置，以JSON字符串形式定义字段类型与属性

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
workspace_name = "tf-workspace-test"
dataspace_name = "tf-dataspace-test"
pipe_name      = "tfpipe01"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="workspace_name=my-workspace"`
2. 环境变量：`export TF_VAR_workspace_name=my-workspace`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建数据管道
4. 运行 `terraform show` 查看已创建的数据管道

## 参考信息

- [华为云安全云脑产品文档](https://support.huaweicloud.com/secmaster/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [安全云脑数据管道最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/pipe)
