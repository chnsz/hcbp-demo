# 部署DataArts Studio工作空间用户

## 应用场景

数据治理中心（DataArts Studio）是华为云提供的一站式数据运营治理平台，工作空间是DataArts Studio中进行数据开发和管理的基本单元。通过为工作空间添加用户并分配角色，可以实现多用户协作开发，并对不同用户进行精细化的权限控制。

本最佳实践适用于在DataArts Studio工作空间中添加IAM用户并分配角色的场景，涵盖工作空间查询、工作空间用户角色查询、IAM用户查询及工作空间用户创建。本最佳实践将介绍如何使用Terraform自动化部署上述资源，实现DataArts Studio工作空间用户管理的Infrastructure as Code。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [DataArts Studio工作空间列表查询数据源（data.huaweicloud_dataarts_studio_workspaces）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_studio_workspaces)
- [DataArts Studio工作空间用户角色列表查询数据源（data.huaweicloud_dataarts_studio_workspace_user_roles）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dataarts_studio_workspace_user_roles)
- [IAM用户列表查询数据源（data.huaweicloud_identity_users）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_users)

### 资源

- [DataArts Studio工作空间用户资源（huaweicloud_dataarts_studio_workspace_user）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dataarts_studio_workspace_user)

### 资源/数据源依赖关系

```
data.huaweicloud_dataarts_studio_workspaces
    ├── data.huaweicloud_dataarts_studio_workspace_user_roles
    └── huaweicloud_dataarts_studio_workspace_user

data.huaweicloud_dataarts_studio_workspace_user_roles
    └── huaweicloud_dataarts_studio_workspace_user

data.huaweicloud_identity_users
    └── huaweicloud_dataarts_studio_workspace_user
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

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下DataArts Studio工作空间信息，用于创建工作空间用户
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

### 3. 查询工作空间用户角色

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询目标工作空间下可用的用户角色信息：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下DataArts Studio工作空间用户角色信息，用于创建工作空间用户
# Query available workspace user roles under the target workspace.
data "huaweicloud_dataarts_studio_workspace_user_roles" "test" {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
}
```

**参数说明**：
- **workspace_id**：工作空间ID，当workspace_id不为空时使用其值，否则引用工作空间查询数据源的返回结果

### 4. 查询IAM用户

在TF文件（如main.tf）中添加以下脚本以告知Terraform按名称查询IAM用户信息。当已通过user_id指定用户ID时可跳过此步骤：

```hcl
variable "user_id" {
  description = "The ID of the IAM user to be added to the workspace"
  type        = string
  default     = ""
  nullable    = false
}

variable "user_name" {
  description = "The name of the IAM user to be added to the workspace"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下IAM用户信息，用于创建工作空间用户
# Query the IAM user by name when user_id is omitted.
data "huaweicloud_identity_users" "test" {
  count = var.user_id == "" ? 1 : 0

  name = var.user_name

  lifecycle {
    precondition {
      condition     = var.user_name != ""
      error_message = "user_name must be provided if user_id is omitted."
    }
  }
}
```

**参数说明**：
- **count**：数据源的创建数，仅当user_id为空时执行IAM用户查询
- **name**：IAM用户名称，通过引用输入变量user_name进行赋值
- **lifecycle.precondition**：前置条件，当user_id为空时必须提供user_name

### 5. 创建DataArts Studio工作空间用户

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建工作空间用户并分配角色：

```hcl
variable "role_ids" {
  description = "The role ID list of the workspace user"
  type        = list(string)
  default     = []
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DataArts Studio工作空间用户资源
# Create a workspace user and assign roles.
resource "huaweicloud_dataarts_studio_workspace_user" "test" {
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(data.huaweicloud_dataarts_studio_workspaces.test[0].workspaces[0].id, "")
  user_id      = var.user_id != "" ? var.user_id : try(data.huaweicloud_identity_users.test[0].users[0].id, "")

  dynamic "roles" {
    for_each = var.role_ids

    content {
      id = roles.value
    }
  }

  lifecycle {
    precondition {
      condition = length([
        for role_id in var.role_ids : role_id
        if contains([for role in data.huaweicloud_dataarts_studio_workspace_user_roles.test.roles : role.id], role_id)
      ]) == length(var.role_ids)
      error_message = "All role_ids must exist in the workspace user roles returned by the data source."
    }
  }
}
```

**参数说明**：
- **workspace_id**：工作空间ID，当workspace_id不为空时使用其值，否则引用工作空间查询数据源的返回结果
- **user_id**：IAM用户ID，当user_id不为空时使用其值，否则引用IAM用户查询数据源的返回结果
- **roles.id**：工作空间用户角色ID，通过引用输入变量role_ids进行赋值
- **lifecycle.precondition**：前置条件，role_ids中的所有角色ID必须存在于工作空间用户角色查询数据源的返回结果中

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`terraform.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# DataArts Studio工作空间
workspace_id = "your-dataarts-studio-workspace-id"

# 工作空间用户
user_name = "your-iam-user-name"
role_ids  = ["r00001"]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="user_name=my-user" -var='role_ids=["r00001"]'`
2. 环境变量：`export TF_VAR_user_name=my-user`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建工作空间用户
4. 运行 `terraform show` 查看已创建的工作空间用户详情

## 参考信息

- [华为云DataArts Studio产品文档](https://support.huaweicloud.com/dataartsstudio/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DataArts Studio工作空间用户最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dataarts/studio-workspace-user)
