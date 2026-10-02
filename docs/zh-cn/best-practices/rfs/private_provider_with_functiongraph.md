# 部署基于FunctionGraph后端的私有Provider

## 应用场景

资源编排服务（Resource Formation Service，RFS）是华为云提供的基础设施即代码（Infrastructure as Code，IaC）服务，支持通过模板化的方式定义、编排和自动化部署云上资源。私有Provider（Private Provider）是RFS提供的一种扩展能力，允许用户将自定义的执行逻辑注册为Provider，从而在模板中调用自定义的资源类型。

函数工作流（FunctionGraph）是一项基于事件驱动的无服务器计算服务，用户无需管理服务器即可运行代码。将FunctionGraph函数作为私有Provider的执行后端，可以把自定义的资源操作逻辑封装为函数，由RFS在编排过程中按需调用，实现灵活的资源扩展。

本最佳实践将介绍如何使用Terraform自动化部署一个基于FunctionGraph后端的RFS私有Provider，包括创建FunctionGraph函数、创建关联该函数的私有Provider，以及为私有Provider发布新版本。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [函数工作流函数（huaweicloud_fgs_function）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/fgs_function)
- [RFS私有Provider（huaweicloud_rfs_private_provider）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_private_provider)
- [RFS私有Provider版本（huaweicloud_rfs_private_provider_version）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_private_provider_version)

### 资源/数据源依赖关系

```
huaweicloud_fgs_function
    ├── huaweicloud_rfs_private_provider
    │       └── huaweicloud_rfs_private_provider_version
    └── huaweicloud_rfs_private_provider_version
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建FunctionGraph函数

在TF文件（如main.tf）中添加以下脚本以创建作为私有Provider执行后端的FunctionGraph函数：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建FunctionGraph函数资源
variable "function_name" {
  description = "The name of the FunctionGraph function"
  type        = string
}

variable "function_app" {
  description = "The group name of the FunctionGraph function"
  type        = string
  default     = "default"
}

variable "function_handler" {
  description = "The handler of the FunctionGraph function"
  type        = string
  default     = "index.handler"
}

variable "function_memory_size" {
  description = "The memory size of the FunctionGraph function in MB"
  type        = number
  default     = 128
}

variable "function_timeout" {
  description = "The timeout of the FunctionGraph function in seconds"
  type        = number
  default     = 3
}

variable "function_runtime" {
  description = "The runtime of the FunctionGraph function"
  type        = string
  default     = "Node.js12.13"
}

variable "function_code" {
  description = "The inline code content of the FunctionGraph function"
  type        = string
}

resource "huaweicloud_fgs_function" "test" {
  name        = var.function_name
  app         = var.function_app
  handler     = var.function_handler
  memory_size = var.function_memory_size
  timeout     = var.function_timeout
  code_type   = "inline"
  runtime     = var.function_runtime
  func_code   = base64encode(var.function_code)
}
```

**参数说明**：
- **name**：函数名称，通过引用输入变量 function_name 进行赋值
- **app**：函数所属应用分组，通过引用输入变量 function_app 进行赋值，默认值为 default
- **handler**：函数执行入口，通过引用输入变量 function_handler 进行赋值，默认值为 index.handler
- **memory_size**：函数内存大小（MB），通过引用输入变量 function_memory_size 进行赋值，默认值为 128
- **timeout**：函数超时时间（秒），通过引用输入变量 function_timeout 进行赋值，默认值为 3
- **code_type**：函数代码类型，此处固定为 inline，表示使用内联代码
- **runtime**：函数运行时环境，通过引用输入变量 function_runtime 进行赋值，默认值为 Node.js12.13
- **func_code**：函数内联代码内容，通过引用输入变量 function_code 进行赋值，并使用 base64encode 进行编码

### 3. 创建RFS私有Provider

在TF文件（如main.tf）中添加以下脚本以创建关联上述FunctionGraph函数的RFS私有Provider：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RFS私有Provider资源
variable "private_provider_name" {
  description = "The name of the RFS private provider"
  type        = string
}

variable "private_provider_description" {
  description = "The description of the RFS private provider"
  type        = string
  default     = ""
}

variable "private_provider_version" {
  description = "The initial version number of the RFS private provider"
  type        = string
  default     = "1.0.0"
}

variable "private_provider_version_description" {
  description = "The initial version description of the RFS private provider"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_private_provider" "test" {
  provider_name        = var.private_provider_name
  function_graph_urn   = huaweicloud_fgs_function.test.urn
  provider_description = var.private_provider_description
  provider_version     = var.private_provider_version
  version_description  = var.private_provider_version_description
}
```

**参数说明**：
- **provider_name**：私有Provider名称，通过引用输入变量 private_provider_name 进行赋值，仅允许小写字母、数字和连字符（-），且在同一域和区域内必须唯一
- **function_graph_urn**：作为执行后端的FunctionGraph函数URN，引用上一步创建的 huaweicloud_fgs_function.test.urn 进行赋值，该参数不支持更新，变更会重建私有Provider
- **provider_description**：私有Provider描述，通过引用输入变量 private_provider_description 进行赋值
- **provider_version**：私有Provider初始版本号，通过引用输入变量 private_provider_version 进行赋值，默认值为 1.0.0，该参数不支持更新
- **version_description**：初始版本描述，通过引用输入变量 private_provider_version_description 进行赋值，该参数不支持更新

### 4. 创建RFS私有Provider版本

在TF文件（如main.tf）中添加以下脚本以为私有Provider发布新版本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RFS私有Provider版本资源
variable "provider_version_number" {
  description = "The version number of the RFS private provider version"
  type        = string
  default     = "2.0.0"
}

variable "provider_version_description" {
  description = "The description of the RFS private provider version"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_private_provider_version" "test" {
  provider_name       = huaweicloud_rfs_private_provider.test.provider_name
  provider_version    = var.provider_version_number
  function_graph_urn  = huaweicloud_fgs_function.test.urn
  version_description = var.provider_version_description
}
```

**参数说明**：
- **provider_name**：私有Provider名称，引用上一步创建的 huaweicloud_rfs_private_provider.test.provider_name 进行赋值
- **provider_version**：新版本号，通过引用输入变量 provider_version_number 进行赋值，默认值为 2.0.0
- **function_graph_urn**：作为执行后端的FunctionGraph函数URN，引用 huaweicloud_fgs_function.test.urn 进行赋值
- **version_description**：版本描述，通过引用输入变量 provider_version_description 进行赋值

> 注意：私有Provider版本的所有参数均不支持更新，变更会重建该版本。

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
function_name         = "tf-test-function"
function_code         = <<EOT
exports.handler = async (event, context) => {
    const result =
    {
        'statusCode': 200,
        'headers':
        {
            'Content-Type': 'application/json'
        },
        'isBase64Encoded': false,
        'body': JSON.stringify(event)
    }
    return result
}
EOT
private_provider_name = "tf-test-provider"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="private_provider_name=my-provider"`
2. 环境变量：`export TF_VAR_private_provider_name=my-provider`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建基于FunctionGraph后端的私有Provider
4. 运行 `terraform show` 查看已创建的基于FunctionGraph后端的私有Provider

## 参考信息

- [华为云资源编排服务（RFS）产品文档](https://support.huaweicloud.com/rfs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RFS基于FunctionGraph后端的私有Provider最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rfs/private-provider-with-functiongraph)
