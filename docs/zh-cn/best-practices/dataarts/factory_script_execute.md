# 部署DataArts Factory脚本执行

## 应用场景

数据治理中心（DataArts Studio）是华为云提供的一站式数据运营治理平台，DataArts Factory（数据开发）模块提供全托管的大数据调度与开发环境，支持DLI SQL等多种脚本类型的创建与执行。通过Factory脚本，用户可以在工作空间中编写和管理数据开发脚本，并关联DLI队列、数据库等资源完成数据处理任务。

本最佳实践适用于在DataArts Studio工作空间中创建DLI SQL脚本并执行一次的场景，涵盖工作空间查询、DLI数据连接查询、DLI数据库与表创建、Factory脚本创建及脚本执行。本最佳实践将介绍如何使用Terraform自动化部署上述资源，实现DataArts Factory脚本开发与执行的Infrastructure as Code管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [DataArts Studio工作空间列表查询数据源（data.huaweicloud_dataarts_studio_workspaces）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_studio_workspaces)
- [DataArts Studio数据连接列表查询数据源（data.huaweicloud_dataarts_studio_data_connections）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_studio_data_connections)

### 资源

- [DLI数据库资源（huaweicloud_dli_database）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_database)
- [DLI表资源（huaweicloud_dli_table）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_table)
- [DataArts Factory脚本资源（huaweicloud_dataarts_factory_script）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_factory_script)
- [DataArts Factory脚本执行资源（huaweicloud_dataarts_factory_script_execute）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_factory_script_execute)

### 资源/数据源依赖关系

```
data.huaweicloud_dataarts_studio_workspaces
    ├── data.huaweicloud_dataarts_studio_data_connections
    ├── huaweicloud_dataarts_factory_script
    └── huaweicloud_dataarts_factory_script_execute

data.huaweicloud_dataarts_studio_data_connections
    └── huaweicloud_dataarts_factory_script

huaweicloud_dli_database
    ├── huaweicloud_dli_table
    └── huaweicloud_dataarts_factory_script

huaweicloud_dli_table
    └── huaweicloud_dataarts_factory_script

huaweicloud_dataarts_factory_script
    └── huaweicloud_dataarts_factory_script_execute
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询DataArts Studio工作空间

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询DataArts Studio工作空间信息。当已通过workspace_id指定工作空间ID时可跳过此步骤：

```hcl
variable "workspace_id" {
  description = "The ID of the workspace"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_id" {
  description = "The ID of the DataArts Studio instance to which the workspace belongs"
  type        = string
  default     = ""
  nullable    = false
}

variable "workspace_name" {
  description = "The name of the workspace used to filter results"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下DataArts Studio工作空间信息，用于创建Factory脚本及执行脚本
# Query workspaces under the specified DataArts Studio instance.
data "huaweicloud_dataarts_studio_workspaces" "test" {
  count = var.workspace_id == "" ? 1 : 0

  instance_id  = var.instance_id
  name         = var.workspace_name != "" ? var.workspace_name : null
  workspace_id = var.workspace_id != "" ? var.workspace_id : null

  lifecycle {
    precondition {
      condition     = var.instance_id != ""
      error_message = "instance_id must be provided if workspace_id is omitted."
    }
  }
}
```

**参数说明**：
- **count**：数据源的创建数，仅当workspace_id为空时执行工作空间查询
- **instance_id**：DataArts Studio实例ID，通过引用输入变量instance_id进行赋值
- **name**：工作空间名称，当workspace_name不为空时用于过滤查询结果
- **workspace_id**：工作空间ID，当workspace_id不为空时用于精确查询
- **lifecycle.precondition**：前置条件，当workspace_id为空时必须提供instance_id

### 3. 查询DLI数据连接

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询DataArts Studio工作空间中的DLI数据连接信息。当已通过connection_name指定连接名称时可跳过此步骤：

```hcl
variable "connection_name" {
  description = "The name of the DLI data connection"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下DataArts Studio工作空间中的DLI数据连接信息，用于创建Factory脚本
# Query data connections in the workspace to obtain the DLI connection name.
data "huaweicloud_dataarts_studio_data_connections" "test" {
  count = var.connection_name == "" ? 1 : 0

  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
  type         = "DLI"
}
```

**参数说明**：
- **count**：数据源的创建数，仅当connection_name为空时执行数据连接查询
- **workspace_id**：工作空间ID，当workspace_id不为空时使用其值，否则引用工作空间查询数据源的返回结果
- **type**：数据连接类型，设置为"DLI"表示DLI数据连接

### 4. 创建DLI数据库

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DLI数据库资源，作为Factory脚本的数据源：

```hcl
variable "dli_database_name" {
  description = "The name of the DLI database created for the script"
  type        = string
}

variable "dli_database_description" {
  description = "The description of the DLI database"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DLI数据库资源，作为Factory脚本的数据源
# Create a DLI database as the data source for the script.
resource "huaweicloud_dli_database" "test" {
  name        = var.dli_database_name
  description = var.dli_database_description
}
```

**参数说明**：
- **name**：DLI数据库名称，通过引用输入变量dli_database_name进行赋值
- **description**：DLI数据库描述，通过引用输入变量dli_database_description进行赋值，默认值为空字符串

### 5. 创建DLI表

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DLI表资源，作为Factory脚本的数据源：

```hcl
variable "dli_table_name" {
  description = "The name of the DLI table created for the script"
  type        = string
}

variable "dli_table_description" {
  description = "The description of the DLI table"
  type        = string
  default     = ""
}

variable "dli_table_columns" {
  description = "The column definitions of the DLI table"
  type = list(object({
    name = string
    type = string
  }))

  default = [
    {
      name = "name"
      type = "string"
    },
    {
      name = "age"
      type = "int"
    },
  ]
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DLI表资源，作为Factory脚本的数据源
# Create a DLI table as the data source for the script.
resource "huaweicloud_dli_table" "test" {
  database_name = huaweicloud_dli_database.test.name
  name          = var.dli_table_name
  data_location = "DLI"
  description   = var.dli_table_description

  dynamic "columns" {
    for_each = var.dli_table_columns

    content {
      name = columns.value.name
      type = columns.value.type
    }
  }
}
```

**参数说明**：
- **database_name**：DLI数据库名称，引用前面创建的DLI数据库资源的名称
- **name**：DLI表名称，通过引用输入变量dli_table_name进行赋值
- **data_location**：数据存储位置，设置为"DLI"
- **description**：DLI表描述，通过引用输入变量dli_table_description进行赋值，默认值为空字符串
- **columns**：表列定义，通过引用输入变量dli_table_columns进行赋值，默认包含name（string）和age（int）两列

### 6. 创建DataArts Factory脚本

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DataArts Factory脚本资源：

```hcl
variable "script_name" {
  description = "The name of the DataArts Factory script"
  type        = string
}

variable "script_type" {
  description = "The type of the DataArts Factory script"
  type        = string
  default     = "DLISQL"
}

variable "script_directory" {
  description = "The directory path where the script is stored in DataArts Factory"
  type        = string
  default     = "/terraform"
}

variable "queue_name" {
  description = "The DLI queue name associated with the script"
  type        = string
  default     = "default"
}

variable "script_description" {
  description = "The description of the DataArts Factory script"
  type        = string
  default     = ""
}

variable "script_configuration" {
  description = "The user-defined configuration parameters of the DataArts Factory script"
  type        = map(string)
  default     = {}
}

variable "script_content" {
  description = "The SQL content of the DataArts Factory script"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DataArts Factory脚本资源
# Create a DataArts Factory script that queries the DLI table.
resource "huaweicloud_dataarts_factory_script" "test" {
  depends_on = [
    huaweicloud_dli_database.test,
    huaweicloud_dli_table.test,
  ]

  workspace_id    = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
  name            = var.script_name
  type            = var.script_type
  connection_name = var.connection_name != "" ? var.connection_name : try(data.huaweicloud_dataarts_studio_data_connections.test[0].connections[0].name, "")
  directory       = var.script_directory
  database        = huaweicloud_dli_database.test.name
  queue_name      = var.queue_name
  description     = var.script_description
  configuration   = var.script_configuration
  content         = var.script_content != "" ? var.script_content : format("SELECT * FROM %s.%s", var.dli_database_name, var.dli_table_name)
}
```

**参数说明**：
- **depends_on**：显式依赖DLI数据库和DLI表资源，确保数据源创建完成后再创建脚本
- **workspace_id**：工作空间ID，当workspace_id不为空时使用其值，否则引用工作空间查询数据源的返回结果
- **name**：脚本名称，通过引用输入变量script_name进行赋值
- **type**：脚本类型，通过引用输入变量script_type进行赋值，默认值为"DLISQL"
- **connection_name**：DLI数据连接名称，当connection_name不为空时使用其值，否则引用数据连接查询数据源的返回结果
- **directory**：脚本存储目录，通过引用输入变量script_directory进行赋值，默认值为"/terraform"
- **database**：关联的DLI数据库名称，引用前面创建的DLI数据库资源的名称
- **queue_name**：关联的DLI队列名称，通过引用输入变量queue_name进行赋值，默认值为"default"
- **description**：脚本描述，通过引用输入变量script_description进行赋值
- **configuration**：用户自定义配置参数，通过引用输入变量script_configuration进行赋值，默认值为空映射
- **content**：脚本SQL内容，当script_content不为空时使用其值，否则自动生成查询DLI表的SELECT语句

### 7. 执行DataArts Factory脚本

在TF文件（如main.tf）中添加以下脚本以告知Terraform执行DataArts Factory脚本：

```hcl
variable "script_execute_params" {
  description = "The execution parameters passed to the script content, in key-value format"
  type        = map(string)
  default = {
    "spark.sql.adaptive.enabled"                                   = "true"
    "spark.sql.adaptive.join.enabled"                              = "true"
    "spark.sql.adaptive.join.skewedJoin.enabled"                   = "true"
    "spark.sql.forcePartitionPredicatesOnPartitionedTable.enabled" = "true"
    "spark.sql.mergeSmallFiles.enabled"                            = "true"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下执行DataArts Factory脚本
# Execute the DataArts Factory script once.
resource "huaweicloud_dataarts_factory_script_execute" "test" {
  depends_on = [
    huaweicloud_dataarts_factory_script.test,
  ]

  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
  script_name  = huaweicloud_dataarts_factory_script.test.name
  params       = length(var.script_execute_params) > 0 ? jsonencode(var.script_execute_params) : null
}
```

**参数说明**：
- **depends_on**：显式依赖Factory脚本资源，确保脚本创建完成后再执行
- **workspace_id**：工作空间ID，当workspace_id不为空时使用其值，否则引用工作空间查询数据源的返回结果
- **script_name**：脚本名称，引用前面创建的Factory脚本资源的名称
- **params**：脚本执行参数，当script_execute_params不为空时通过jsonencode编码为JSON字符串，否则为null

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`terraform.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# DataArts Studio实例与工作空间
workspace_id = "your-dataarts-studio-workspace-id"

# DLI数据库与表
dli_database_name = "tf_test_database"
dli_table_name    = "tf_test_table"

# DataArts Factory脚本
script_name = "tf_test_factory_script"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="script_name=my-script" -var="dli_database_name=my-db"`
2. 环境变量：`export TF_VAR_script_name=my-script`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DataArts Factory脚本并执行
4. 运行 `terraform show` 查看已创建的DataArts Factory脚本及执行详情

## 参考信息

- [华为云DataArts Studio产品文档](https://support.huaweicloud.com/dataartsstudio/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DataArts Factory脚本执行最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dataarts/factory-script-execute)
