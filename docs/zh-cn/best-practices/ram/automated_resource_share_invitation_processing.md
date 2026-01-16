# 部署自动化资源分享邀请处理

## 应用场景

资源访问管理（Resource Access Manager，RAM）是华为云提供的资源共享服务，支持跨账号的资源共享和管理，帮助您实现资源的统一管理和访问控制。通过自动化资源分享邀请处理功能，可以批量查询待处理的资源分享邀请，并自动接受或拒绝这些邀请，提高资源分享管理的效率和便捷性。本最佳实践将介绍如何使用Terraform自动化处理资源分享邀请，包括查询待处理的邀请和批量接受或拒绝邀请。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [资源分享邀请查询数据源（data.huaweicloud_ram_resource_share_invitations）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ram_resource_share_invitations)

### 资源

- [资源分享接受者资源（huaweicloud_ram_resource_share_accepter）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ram_resource_share_accepter)

### 资源/数据源依赖关系

```text
data.huaweicloud_ram_resource_share_invitations
    └── huaweicloud_ram_resource_share_accepter
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询待处理的资源分享邀请

在TF文件（如main.tf）中添加以下脚本以查询待处理的资源分享邀请：

```hcl
variable "resource_share_ids" {
  description = "List of resource share IDs to query invitations for"
  type        = list(string)
  default     = []
}

# 查询指定资源分享ID列表的待处理邀请
data "huaweicloud_ram_resource_share_invitations" "test" {
  resource_share_ids = var.resource_share_ids
  status             = "pending"
}
```

**参数说明**：
- **resource_share_ids**：资源分享ID列表，通过引用输入变量 `resource_share_ids` 进行赋值，用于指定要查询的资源分享
- **status**：邀请状态，设置为 `pending` 表示查询待处理的邀请

### 3. 处理资源分享邀请

在TF文件（如main.tf）中添加以下脚本以处理资源分享邀请：

```hcl
variable "action" {
  description = "The action to perform on invitations"
  type        = string
  default     = "reject"

  validation {
    condition     = contains(["accept", "reject"], var.action)
    error_message = "The action must be either 'accept' or 'reject'."
  }
}

# 批量处理邀请：接受或拒绝
resource "huaweicloud_ram_resource_share_accepter" "test" {
  count = length(data.huaweicloud_ram_resource_share_invitations.test.resource_share_invitations)

  resource_share_invitation_id = data.huaweicloud_ram_resource_share_invitations.test.resource_share_invitations[count.index].id
  action                       = var.action
}
```

**参数说明**：
- **count**：资源数量，根据查询到的待处理邀请数量动态创建资源
- **resource_share_invitation_id**：资源分享邀请ID，通过引用数据源查询结果中的邀请ID进行赋值
- **action**：处理动作，通过引用输入变量 `action` 进行赋值，可选值为 `accept`（接受）或 `reject`（拒绝），默认为 `reject`

> 注意：通过 `count` 参数可以根据查询到的邀请数量动态创建资源，实现批量处理。处理动作可以是接受或拒绝，根据实际需求进行配置。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 资源分享ID列表（必填）
resource_share_ids = [
  "resource-share-id-1",
  "resource-share-id-2"
]

# 处理动作（可选，默认为reject）
action = "accept"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="resource_share_ids=['resource-share-id-1','resource-share-id-2']" -var="action=accept"`
2. 环境变量：`export TF_VAR_resource_share_ids='["resource-share-id-1","resource-share-id-2"]'`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来处理资源分享邀请：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源处理计划
3. 确认资源计划无误后，运行 `terraform apply` 开始处理资源分享邀请
4. 运行 `terraform show` 查看已处理的邀请

## 参考信息

- [华为云RAM产品文档](https://support.huaweicloud.com/ram/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RAM自动化资源分享邀请处理最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ram/automated-resource-share-invitation-processing)
