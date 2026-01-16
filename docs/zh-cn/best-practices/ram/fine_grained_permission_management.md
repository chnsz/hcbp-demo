# 部署细粒度权限管理

## 应用场景

资源访问管理（Resource Access Manager，RAM）是华为云提供的资源共享服务，支持跨账号的资源共享和管理，帮助您实现资源的统一管理和访问控制。通过细粒度权限管理功能，可以为资源分享配置精细的权限控制，实现更灵活和安全的资源共享。通过查询可用的权限并关联到资源分享，可以精确控制共享资源的使用范围和操作权限。本最佳实践将介绍如何使用Terraform自动化部署细粒度权限管理，包括查询可用权限和将权限关联到资源分享。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [资源权限查询数据源（data.huaweicloud_ram_resource_permissions）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ram_resource_permissions)

### 资源

- [资源分享权限资源（huaweicloud_ram_resource_share_permission）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ram_resource_share_permission)

### 资源/数据源依赖关系

```text
data.huaweicloud_ram_resource_permissions
    └── huaweicloud_ram_resource_share_permission
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用的资源权限

在TF文件（如main.tf）中添加以下脚本以查询可用的资源权限：

```hcl
variable "query_resource_type" {
  description = "The resource type for querying available permissions"
  type        = string
  default     = ""
}

variable "query_permission_type" {
  description = "The type of the permission to query"
  type        = string
  default     = "ALL"
}

variable "query_permission_name" {
  description = "The name of the permission to query"
  type        = string
  default     = ""
}

# 查询可用的资源权限
data "huaweicloud_ram_resource_permissions" "test" {
  resource_type   = var.query_resource_type != "" ? var.query_resource_type : null
  permission_type = var.query_permission_type
  name            = var.query_permission_name != "" ? var.query_permission_name : null
}
```

**参数说明**：
- **resource_type**：资源类型，通过引用输入变量 `query_resource_type` 进行赋值，用于指定要查询权限的资源类型，可选参数
- **permission_type**：权限类型，通过引用输入变量 `query_permission_type` 进行赋值，可选值包括 `ALL`（全部）、`SYSTEM`（系统权限）、`CUSTOM`（自定义权限），默认为 `ALL`
- **name**：权限名称，通过引用输入变量 `query_permission_name` 进行赋值，用于按名称过滤权限，可选参数

> 注意：通过设置不同的查询条件，可以筛选出符合要求的权限列表。如果不指定资源类型，将查询所有资源类型的权限。

### 3. 将权限关联到资源分享

在TF文件（如main.tf）中添加以下脚本以将权限关联到资源分享：

```hcl
variable "resource_share_id" {
  description = "The ID of the RAM resource share"
  type        = string
}

variable "permission_replace" {
  description = "Whether to replace existing permissions when associating a new permission"
  type        = bool
  default     = false
}

# 批量将查询到的权限关联到资源分享
resource "huaweicloud_ram_resource_share_permission" "test" {
  count = length(data.huaweicloud_ram_resource_permissions.test.permissions)

  resource_share_id = var.resource_share_id
  permission_id     = data.huaweicloud_ram_resource_permissions.test.permissions[count.index].id
  replace           = var.permission_replace
}
```

**参数说明**：
- **count**：资源数量，根据查询到的权限数量动态创建资源
- **resource_share_id**：资源分享ID，通过引用输入变量 `resource_share_id` 进行赋值，用于指定要关联权限的资源分享
- **permission_id**：权限ID，通过引用数据源查询结果中的权限ID进行赋值
- **replace**：是否替换现有权限，通过引用输入变量 `permission_replace` 进行赋值，当关联新权限时是否替换现有权限，默认为 `false`

> 注意：通过 `count` 参数可以根据查询到的权限数量动态创建资源，实现批量关联权限。如果 `replace` 为 `true`，关联新权限时会替换资源分享的现有权限；如果为 `false`，则追加新权限。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 权限查询配置（可选）
query_resource_type = "vpc:subnets"

# 资源分享ID（必填）
resource_share_id = "your-resource-share-id"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="query_resource_type=vpc:subnets" -var="resource_share_id=your-resource-share-id"`
2. 环境变量：`export TF_VAR_resource_share_id=your-resource-share-id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始关联权限到资源分享
4. 运行 `terraform show` 查看已关联的权限

## 参考信息

- [华为云RAM产品文档](https://support.huaweicloud.com/ram/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RAM细粒度权限管理最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ram/fine-grained-permission-management)
