# 部署用户组与策略关联

## 应用场景

统一身份认证服务（Identity and Access Management，IAM）是华为云提供的基础身份认证与访问管理服务，为华为云用户提供身份管理、权限管理和访问控制等核心功能。通过将策略关联到用户组，可以为用户组中的用户统一授予权限，实现基于用户组的权限管理。本最佳实践将介绍如何使用Terraform自动化部署IAM用户组与策略关联，包括查询IAM策略、创建用户组和将策略关联到用户组。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [IAM策略数据源（huaweicloud_identityv5_policies）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identityv5_policies)

### 资源

- [IAM用户组资源（huaweicloud_identityv5_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identityv5_group)
- [IAM策略与用户组关联资源（huaweicloud_identityv5_policy_group_attach）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identityv5_policy_group_attach)

### 资源/数据源依赖关系

```text
data.huaweicloud_identityv5_policies
    └── huaweicloud_identityv5_policy_group_attach

huaweicloud_identityv5_group
    └── huaweicloud_identityv5_policy_group_attach
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询IAM策略数据源

在TF文件（如main.tf）中添加以下脚本以查询IAM策略信息：

```hcl
variable "policy_type" {
  description = "The type of the policy"
  type        = string
  default     = "system"
}

variable "policy_names" {
  description = "The name list of policies to be associated with the user group"
  type        = list(string)
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的IAM策略信息，用于关联到用户组
data "huaweicloud_identityv5_policies" "test" {
  policy_type = var.policy_type
}

# 根据策略名称过滤策略列表
locals {
  filtered_policies = [for policy in data.huaweicloud_identityv5_policies.test.policies : policy if contains(var.policy_names, policy.policy_name)]
}
```

**参数说明**：
- **policy_type**：策略类型，通过引用输入变量policy_type进行赋值，默认值为"system"（系统策略）
- **policy_names**：策略名称列表，通过引用输入变量policy_names进行赋值，用于过滤需要关联的策略
- **locals.filtered_policies**：根据策略名称过滤后的策略列表，用于后续关联到用户组

### 3. 创建IAM用户组资源

在TF文件（如main.tf）中添加以下脚本以创建IAM用户组：

```hcl
variable "group_name" {
  description = "The name of the user group"
  type        = string
}

variable "group_description" {
  description = "The description of the user group"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IAM用户组资源
resource "huaweicloud_identityv5_group" "test" {
  group_name  = var.group_name
  description = var.group_description
}
```

**参数说明**：
- **group_name**：用户组名称，通过引用输入变量group_name进行赋值
- **description**：用户组描述，通过引用输入变量group_description进行赋值，可选参数，默认值为空字符串

### 4. 创建IAM策略与用户组关联资源

在TF文件（如main.tf）中添加以下脚本以将策略关联到用户组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IAM策略与用户组关联资源
resource "huaweicloud_identityv5_policy_group_attach" "test" {
  count = length(local.filtered_policies)

  policy_id = try(local.filtered_policies[count.index].policy_id, null)
  group_id  = huaweicloud_identityv5_group.test.id
}
```

**参数说明**：
- **count**：资源创建数量，根据过滤后的策略列表长度动态创建关联资源
- **policy_id**：策略ID，通过引用过滤后的策略列表进行赋值
- **group_id**：用户组ID，通过引用用户组资源进行赋值

> 注意：策略与用户组关联资源使用count参数，根据过滤后的策略列表动态创建多个关联资源。确保策略名称列表中的策略在IAM中存在，否则过滤后的策略列表可能为空，导致无法创建关联。

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# IAM用户组配置
group_name        = "tf_test_group"
group_description = "Test user group"

# IAM策略配置
policy_type  = "system"
policy_names = [
  "ModelArtsFullAccessPolicy",
  "SCMReadOnlyPolicy"
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `group_name`需要设置用户组名称
   - `group_description`可以设置用户组描述信息，可选参数
   - `policy_type`可以设置为"system"（系统策略）或其他策略类型
   - `policy_names`需要设置为要关联的策略名称列表，确保这些策略在IAM中存在
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="group_name=my-group" -var='policy_names=["Policy1","Policy2"]'`
2. 环境变量：`export TF_VAR_group_name=my-group` 和 `export TF_VAR_policy_names='["Policy1","Policy2"]'`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。确保策略名称列表中的策略在IAM中存在，否则过滤后的策略列表可能为空，导致无法创建关联。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建IAM用户组与策略关联：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建IAM用户组和策略关联
4. 运行 `terraform show` 查看已创建的IAM用户组和策略关联详情

> 注意：策略与用户组关联创建后，用户组中的用户将获得关联策略所授予的权限。确保策略名称正确，否则过滤后的策略列表可能为空，导致无法创建关联。建议在创建关联前先查询IAM中可用的策略列表，确认策略名称是否正确。

## 参考信息

- [华为云IAM产品文档](https://support.huaweicloud.com/iam/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [用户组与策略关联最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/iam/v5/group-policies-associate)
