# 部署组织

## 应用场景

华为云组织（Organizations）服务是华为云提供的多账户管理服务，支持企业通过组织架构统一管理多个华为云账户，实现资源的集中管理和权限的统一控制。组织是Organizations服务的顶层资源，用于创建多账户架构的根节点，并支持启用策略类型、为根组织配置标签等能力。

本最佳实践适用于需要从零创建Organizations组织、启用服务控制策略（SCP）等策略类型并为根组织设置标签的场景。本最佳实践将介绍如何使用Terraform自动化部署Organizations组织，包括组织的创建及策略类型、标签等配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [Organizations组织资源（huaweicloud_organizations_organization）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/organizations_organization)

### 资源/数据源依赖关系

```
huaweicloud_organizations_organization
```

> 注意：Organizations组织是独立资源，不依赖其他资源或数据源。每个华为云账户仅可创建一个组织，创建后可通过该组织继续创建组织单元和账户。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建Organizations组织

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建Organizations组织资源：

```hcl
variable "enabled_policy_types" {
  description = "The list of Organizations policy types to enable in the Organization Root"
  type        = list(string)
}

variable "root_tags" {
  description = "The key/value to attach to the root"
  type        = map(string)
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Organizations组织资源
resource "huaweicloud_organizations_organization" "test" {
  enabled_policy_types = var.enabled_policy_types
  root_tags            = var.root_tags
}
```

**参数说明**：
- **enabled_policy_types**：在组织根节点启用的策略类型列表，通过引用输入变量enabled_policy_types进行赋值，例如可设置为["service_control_policy"]以启用服务控制策略
- **root_tags**：附加到组织根节点的标签，通过引用输入变量root_tags进行赋值，用于对根组织进行分类和管理

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`terraform.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 组织配置
enabled_policy_types = ["service_control_policy"]

root_tags = {
  key   = "value"
  owner = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var='enabled_policy_types=["service_control_policy"]'`
2. 环境变量：`export TF_VAR_enabled_policy_types='["service_control_policy"]'`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建组织
4. 运行 `terraform show` 查看已创建的组织详情

## 参考信息

- [华为云Organizations产品文档](https://support.huaweicloud.com/organizations/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Organizations组织最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/organizations/organization)
