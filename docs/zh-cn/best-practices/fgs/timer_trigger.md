# 部署定时触发器

## 应用场景

函数工作流（FunctionGraph）的定时触发器（Timer Trigger）是一种基于时间调度的触发器类型，支持Cron表达式和固定间隔两种调度方式。通过定时触发器，您可以实现定时任务、周期性数据处理、定时备份、定时监控等功能。

定时触发器特别适用于需要按固定时间间隔或特定时间点执行的任务，如数据清理、报表生成、系统监控、定时通知等场景。本最佳实践将介绍如何使用Terraform自动化部署一个带有定时触发器的FunctionGraph函数。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

本最佳实践未使用数据源。

### 资源

- [FunctionGraph函数资源（huaweicloud_fgs_function）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/fgs_function)
- [FunctionGraph函数触发器资源（huaweicloud_fgs_function_trigger）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/fgs_function_trigger)

### 资源/数据源依赖关系

```
huaweicloud_fgs_function
    └── huaweicloud_fgs_function_trigger
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../docs/introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建FunctionGraph函数

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建FunctionGraph函数资源：

```hcl
# Variable definitions for FunctionGraph resources
variable "function_name" {
  description = "The name of the FunctionGraph function"
  type        = string
}

variable "function_memory_size" {
  description = "The memory size of the function in MB"
  type        = number
  default     = 128
}

variable "function_timeout" {
  description = "The timeout of the function in seconds"
  type        = number
  default     = 10
}

variable "function_runtime" {
  description = "The runtime of the function"
  type        = string
  default     = "Python2.7"
}

variable "function_code" {
  description = "The source code of the function"
  type        = string
  default     = <<EOT
import json

def handler(event, context):
    print("FunctionGraph timer trigger executed!")
    print("Event:", json.dumps(event))
    print("Context:", json.dumps(context))
    return {
        'statusCode': 200,
        'body': json.dumps('Hello, FunctionGraph!')
    }
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
  app         = "default"
  handler     = "index.handler"
  memory_size = var.function_memory_size
  timeout     = var.function_timeout
  runtime     = var.function_runtime
  code_type   = "inline"
  func_code   = base64encode(var.function_code)
  description = var.function_description
}
```

**参数说明**：
- **name**：FunctionGraph函数的名称，通过引用输入变量function_name进行赋值
- **app**：函数所属的应用名称，设置为"default"表示使用默认应用
- **handler**：函数的入口点，设置为"index.handler"表示处理函数在index.py文件中的handler方法
- **memory_size**：函数的内存大小（MB），通过引用输入变量function_memory_size进行赋值，默认值为128MB
- **timeout**：函数的超时时间（秒），通过引用输入变量function_timeout进行赋值，默认值为10秒
- **runtime**：函数的运行时环境，通过引用输入变量function_runtime进行赋值，默认值为Python2.7
- **code_type**：代码类型，设置为"inline"表示内联代码
- **func_code**：函数的源代码，通过base64编码输入变量function_code进行赋值
- **description**：函数的描述信息，通过引用输入变量function_description进行赋值

### 3. 创建FunctionGraph定时触发器

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建FunctionGraph定时触发器资源：

```hcl
# Variable definitions for timer trigger
variable "trigger_status" {
  description = "The status of the FunctionGraph timer trigger"
  type        = string
  default     = "ACTIVE"
}

variable "trigger_name" {
  description = "The name of the FunctionGraph timer trigger"
  type        = string
}

variable "trigger_schedule_type" {
  description = "The schedule type of the FunctionGraph timer trigger"
  type        = string
  default     = "Cron"
}

variable "trigger_sync_execution" {
  description = "Whether to execute the function synchronously"
  type        = bool
  default     = false
}

variable "trigger_user_event" {
  description = "The user event description for the FunctionGraph timer trigger"
  type        = string
  default     = ""
}

variable "trigger_schedule" {
  description = "The schedule expression for the FunctionGraph timer trigger"
  type        = string
  default     = "@every 1h30m"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建FunctionGraph定时触发器资源
resource "huaweicloud_fgs_function_trigger" "test" {
  function_urn = huaweicloud_fgs_function.test.urn
  type         = "TIMER"
  status       = var.trigger_status
  event_data   = jsonencode({
    "name"           = var.trigger_name
    "schedule_type"  = var.trigger_schedule_type
    "sync_execution" = var.trigger_sync_execution
    "user_event"     = var.trigger_user_event
    "schedule"       = var.trigger_schedule
  })
}
```

**参数说明**：
- **function_urn**：触发器关联的FunctionGraph函数的URN，通过引用huaweicloud_fgs_function.test.urn进行赋值
- **type**：触发器类型，设置为"TIMER"表示定时触发器
- **status**：触发器的状态，通过引用输入变量trigger_status进行赋值，默认值为"ACTIVE"表示激活状态
- **event_data**：触发器的事件数据，以JSON格式包含以下参数：
  - **name**：触发器的名称，通过引用输入变量trigger_name进行赋值
  - **schedule_type**：调度类型，通过引用输入变量trigger_schedule_type进行赋值，默认值为"Cron"表示使用Cron表达式
  - **sync_execution**：是否同步执行，通过引用输入变量trigger_sync_execution进行赋值，默认值为false表示异步执行
  - **user_event**：用户事件描述，通过引用输入变量trigger_user_event进行赋值
  - **schedule**：调度表达式，通过引用输入变量trigger_schedule进行赋值，支持Cron表达式和固定间隔格式

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 函数基本信息
function_name        = "tf_test_timer_function"
function_description = "Created by Terraform for timer trigger example"

# 触发器配置
trigger_name       = "tf_test_timer_cron"
trigger_user_event = "Timer trigger with Cron schedule type, triggered every three days"
trigger_schedule   = "@every 3d"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="function_name=my-function" -var="trigger_name=my-trigger"`
2. 环境变量：`export TF_VAR_function_name=my-function`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建FunctionGraph函数和定时触发器
4. 运行 `terraform show` 查看已创建的FunctionGraph函数和定时触发器

## 参考信息

- [华为云FunctionGraph产品文档](https://support.huaweicloud.com/functiongraph/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [FunctionGraph定时触发器最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/fgs/triggers/timer)
