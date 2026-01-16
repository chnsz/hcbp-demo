# 部署工作流版本

## 应用场景

安全云脑（SecMaster）是华为云提供的安全态势感知与安全运营平台，支持安全事件的统一管理、分析和响应，帮助您实现安全运营的自动化和智能化。通过工作流版本功能，可以为工作流创建不同版本，实现工作流的版本管理和迭代更新。工作流版本包含工作流拓扑图的Base64编码和参数配置，支持JSON格式的任务流定义。本最佳实践将介绍如何使用Terraform自动化部署工作流版本，包括工作空间查询、工作流查询和工作流版本的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [工作空间查询数据源（data.huaweicloud_secmaster_workspaces）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/secmaster_workspaces)
- [工作流查询数据源（data.huaweicloud_secmaster_workflows）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/secmaster_workflows)

### 资源

- [工作流版本资源（huaweicloud_secmaster_workflow_version）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_workflow_version)

### 资源/数据源依赖关系

```text
data.huaweicloud_secmaster_workspaces
    └── data.huaweicloud_secmaster_workflows
        └── huaweicloud_secmaster_workflow_version
```

> 注意：工作空间和工作流的查询是可选的。如果提供了 `workspace_id` 和 `workflow_id`，则直接使用这些ID；否则通过名称查询对应的ID。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询工作空间（可选）

在TF文件（如main.tf）中添加以下脚本以查询工作空间（可选）：

```hcl
variable "workspace_id" {
  description = "The ID of the workspace"
  type        = string
  default     = ""
  nullable    = false
}

variable "workspace_name" {
  description = "The name of the workspace"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.workspace_id != "" || var.workspace_name != ""
    error_message = "At least one of workspace_id and workspace_name must be provided."
  }
}

# 查询工作空间（当workspace_id为空时通过名称查询）
data "huaweicloud_secmaster_workspaces" "test" {
  count = var.workspace_id == "" ? 1 : 0

  name = var.workspace_name
}
```

**参数说明**：
- **count**：数据源数量，当 `workspace_id` 为空时创建数据源进行查询
- **name**：工作空间名称，通过引用输入变量 `workspace_name` 进行赋值

> 注意：如果提供了 `workspace_id`，则不需要查询工作空间；如果只提供了 `workspace_name`，则需要通过数据源查询工作空间ID。

### 3. 查询工作流（可选）

在TF文件（如main.tf）中添加以下脚本以查询工作流（可选）：

```hcl
variable "workflow_id" {
  description = "The ID of the workflow"
  type        = string
  default     = ""
  nullable    = false
}

variable "workflow_name" {
  description = "The name of the workflow"
  type        = string
}

# 查询工作流（当workflow_id为空时通过名称查询）
data "huaweicloud_secmaster_workflows" "test" {
  count = var.workflow_id == "" ? 1 : 0

  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  name         = var.workflow_name
}
```

**参数说明**：
- **count**：数据源数量，当 `workflow_id` 为空时创建数据源进行查询
- **workspace_id**：工作空间ID，优先使用输入的 `workspace_id`，如果为空则使用查询到的工作空间ID
- **name**：工作流名称，通过引用输入变量 `workflow_name` 进行赋值

> 注意：如果提供了 `workflow_id`，则不需要查询工作流；如果只提供了 `workflow_name`，则需要通过数据源查询工作流ID。

### 4. 创建工作流版本

在TF文件（如main.tf）中添加以下脚本以创建工作流版本：

```hcl
variable "workflow_version_taskflow" {
  description = "The Base64 encoded of the workflow topology diagram"
  type        = string
}

variable "workflow_version_taskconfig" {
  description = "The parameters configuration of the workflow topology diagram"
  type        = string
}

variable "workflow_version_taskflow_type" {
  description = "The taskflow type of the workflow"
  type        = string
  default     = "JSON"
}

variable "workflow_version_aop_type" {
  description = "The aop type of the workflow"
  type        = string
  default     = "NORMAL"
}

variable "workflow_version_description" {
  description = "The description of the workflow version"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建工作流版本资源
resource "huaweicloud_secmaster_workflow_version" "test" {
  workspace_id  = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_secmaster_workspaces.test[0].workspaces[0].id, null)
  workflow_id    = var.workflow_id != "" ? var.workflow_id : try(data.huaweicloud_secmaster_workflows.test[0].workflows[0].id, null)
  name           = var.workflow_name
  taskflow       = var.workflow_version_taskflow
  taskconfig     = var.workflow_version_taskconfig
  taskflow_type  = var.workflow_version_taskflow_type
  aop_type       = var.workflow_version_aop_type
  description    = var.workflow_version_description
}
```

**参数说明**：
- **workspace_id**：工作空间ID，优先使用输入的 `workspace_id`，如果为空则使用查询到的工作空间ID
- **workflow_id**：工作流ID，优先使用输入的 `workflow_id`，如果为空则使用查询到的工作流ID
- **name**：工作流名称，通过引用输入变量 `workflow_name` 进行赋值
- **taskflow**：工作流拓扑图的Base64编码，通过引用输入变量 `workflow_version_taskflow` 进行赋值
- **taskconfig**：工作流拓扑图的参数配置，通过引用输入变量 `workflow_version_taskconfig` 进行赋值
- **taskflow_type**：任务流类型，通过引用输入变量 `workflow_version_taskflow_type` 进行赋值，默认为 `JSON`
- **aop_type**：AOP类型，通过引用输入变量 `workflow_version_aop_type` 进行赋值，默认为 `NORMAL`
- **description**：工作流版本描述，通过引用输入变量 `workflow_version_description` 进行赋值，可选参数

> 注意：工作流拓扑图需要以Base64编码的形式提供。任务流类型通常为JSON格式，AOP类型默认为NORMAL。

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 工作空间和工作流信息（必填）
workspace_name = "tf_test_workspace"
workflow_name  = "tf_test_workflow"

# 工作流版本配置（必填）
workflow_version_taskflow    = "your_workflow_taskflow"
workflow_version_taskconfig  = "your_workflow_taskconfig"
workflow_version_description = "This is a workflow version created by Terraform"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="workspace_name=tf_test_workspace" -var="workflow_name=tf_test_workflow"`
2. 环境变量：`export TF_VAR_workspace_name=tf_test_workspace`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建工作流版本
4. 运行 `terraform show` 查看已创建的工作流版本

## 参考信息

- [华为云SecMaster产品文档](https://support.huaweicloud.com/secmaster/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SecMaster工作流版本最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/workflow-version)
