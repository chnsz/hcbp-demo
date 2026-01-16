# 部署工作空间

## 应用场景

安全云脑（SecMaster）是华为云提供的安全态势感知与安全运营平台，支持安全事件的统一管理、分析和响应，帮助您实现安全运营的自动化和智能化。工作空间是安全云脑的基础资源，用于隔离和管理不同业务场景的安全资源。通过创建工作空间，可以在指定的项目下创建独立的安全运营环境，实现安全资源的统一管理和隔离。本最佳实践将介绍如何使用Terraform自动化部署工作空间，包括工作空间基本信息、项目配置、企业项目配置和标签配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [工作空间资源（huaweicloud_secmaster_workspace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_workspace)

### 资源/数据源依赖关系

```text
huaweicloud_secmaster_workspace
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建工作空间

在TF文件（如main.tf）中添加以下脚本以创建工作空间：

```hcl
variable "workspace_name" {
  description = "The name of the workspace"
  type        = string
}

variable "workspace_project_name" {
  description = "The name of the project to in which to create the workspace"
  type        = string
}

variable "workspace_description" {
  description = "The description of the workspace"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the workspace belongs"
  type        = string
  default     = null
}

variable "workspace_tags" {
  description = "The key/value pairs to associate with the workspace"
  type        = map(string)
  default     = {}
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建工作空间资源
resource "huaweicloud_secmaster_workspace" "test" {
  name                  = var.workspace_name
  project_name          = var.workspace_project_name
  description           = var.workspace_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.workspace_tags
}
```

**参数说明**：
- **name**：工作空间名称，通过引用输入变量 `workspace_name` 进行赋值
- **project_name**：项目名称，通过引用输入变量 `workspace_project_name` 进行赋值，用于指定创建工作空间的项目
- **description**：工作空间描述，通过引用输入变量 `workspace_description` 进行赋值，可选参数
- **enterprise_project_id**：企业项目ID，通过引用输入变量 `enterprise_project_id` 进行赋值，用于指定工作空间所属的企业项目，可选参数
- **tags**：标签，通过引用输入变量 `workspace_tags` 进行赋值，用于为工作空间添加键值对标签，可选参数

> 注意：工作空间必须在指定的项目下创建。企业项目ID用于实现资源的统一管理和隔离，如果不指定则使用默认企业项目。标签可以用于资源的分类和管理。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 工作空间基本信息（必填）
workspace_name         = "tf_test_workspace"
workspace_project_name = "cn-north-4"
workspace_description  = "This is a SecMaster workspace created by Terraform"

# 企业项目配置（可选）
enterprise_project_id = "0"

# 标签配置（可选）
workspace_tags = {
  owner = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="workspace_name=tf_test_workspace" -var="workspace_project_name=cn-north-4"`
2. 环境变量：`export TF_VAR_workspace_name=tf_test_workspace`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建工作空间
4. 运行 `terraform show` 查看已创建的工作空间

## 参考信息

- [华为云SecMaster产品文档](https://support.huaweicloud.com/secmaster/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SecMaster工作空间最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/workspace)
