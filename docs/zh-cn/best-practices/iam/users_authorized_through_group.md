# 部署通过用户组授权的用户

## 应用场景

统一身份认证服务（Identity and Access Management, IAM）是华为云提供的基础身份认证与访问管理服务，通过用户、用户组、角色和策略的灵活组合，实现细粒度的权限控制。通过用户组授权用户是一种常见的权限管理方式，可以简化权限管理流程，提高管理效率。

本最佳实践将介绍如何使用Terraform自动化部署IAM角色、用户组和用户，并通过用户组为用户授权。通过这种方式，您可以实现基于用户组的权限管理，当需要为多个用户授予相同权限时，只需将用户添加到相应的用户组，即可自动继承用户组的权限，无需为每个用户单独配置权限。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [IAM角色查询数据源（data.huaweicloud_identity_role）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_role)
- [IAM项目查询数据源（data.huaweicloud_identity_projects）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### 资源

- [IAM角色资源（huaweicloud_identity_role）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identity_role)
- [IAM用户组资源（huaweicloud_identity_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identity_group)
- [IAM角色分配资源（huaweicloud_identity_role_assignment）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identity_role_assignment)
- [随机密码资源（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [IAM用户资源（huaweicloud_identity_user）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identity_user)
- [IAM用户组成员关系资源（huaweicloud_identity_group_membership）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identity_group_membership)

### 资源/数据源依赖关系

```
data.huaweicloud_identity_role.test
    └── huaweicloud_identity_role_assignment.test

huaweicloud_identity_role.test
    └── huaweicloud_identity_role_assignment.test

data.huaweicloud_identity_projects.test
    └── huaweicloud_identity_role_assignment.test

huaweicloud_identity_group.test
    ├── huaweicloud_identity_role_assignment.test
    └── huaweicloud_identity_group_membership.test

random_password.test
    └── huaweicloud_identity_user.test

huaweicloud_identity_user.test
    └── huaweicloud_identity_group_membership.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询IAM角色信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建IAM角色分配资源：

```hcl
variable "role_id" {
  description = "IAM角色的ID"
  type        = string
  default     = ""
  nullable    = false
}

variable "role_policy" {
  description = "IAM角色的策略"
  type        = string
  default     = ""
  nullable    = false
}

variable "role_name" {
  description = "IAM角色的名称"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = !(var.role_name == "" && var.role_id == "")
    error_message = "当role_id未提供时，必须提供role_name。"
  }
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下符合条件的IAM角色信息，用于创建IAM角色分配资源
data "huaweicloud_identity_role" "test" {
  count = var.role_id == "" && var.role_policy == "" ? 1 : 0

  name = var.role_name
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行IAM角色查询数据源，仅当 `var.role_id` 为空且 `var.role_policy` 为空时创建数据源（即执行IAM角色查询）
- **name**：IAM角色的名称，通过引用输入变量 `role_name` 进行赋值

### 3. 创建IAM角色资源

在TF文件中添加以下脚本以告知Terraform创建IAM角色资源：

```hcl
variable "role_type" {
  description = "IAM角色的类型"
  type        = string
  default     = "XA"
}

variable "role_description" {
  description = "IAM角色的描述"
  type        = string
  default     = ""

  validation {
    condition     = !(var.role_description == "" && var.role_policy != "")
    error_message = "当role_policy提供时，必须提供role_description。"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IAM角色资源
resource "huaweicloud_identity_role" "test" {
  count = var.role_id == "" && var.role_policy != "" ? 1 : 0

  name        = var.role_name
  type        = var.role_type
  description = var.role_description
  policy      = var.role_policy
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建IAM角色资源，仅当 `var.role_id` 为空且 `var.role_policy` 不为空时创建资源
- **name**：IAM角色的名称，通过引用输入变量 `role_name` 进行赋值
- **type**：IAM角色的类型，通过引用输入变量 `role_type` 进行赋值，默认为"XA"表示自定义角色
- **description**：IAM角色的描述，通过引用输入变量 `role_description` 进行赋值
- **policy**：IAM角色的策略，通过引用输入变量 `role_policy` 进行赋值，策略为JSON格式的字符串

### 4. 创建IAM用户组资源

在TF文件中添加以下脚本以告知Terraform创建IAM用户组资源：

```hcl
variable "group_id" {
  description = "IAM用户组的ID"
  type        = string
  default     = ""
  nullable    = false
}

variable "group_name" {
  description = "IAM用户组的名称"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = !(var.group_name == "" && var.group_id == "")
    error_message = "当group_id未提供时，必须提供group_name。"
  }
}

variable "group_description" {
  description = "IAM用户组的描述"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IAM用户组资源
resource "huaweicloud_identity_group" "test" {
  count = var.group_id == "" ? 1 : 0

  name        = var.group_name
  description = var.group_description
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建IAM用户组资源，仅当 `var.group_id` 为空时创建资源
- **name**：IAM用户组的名称，通过引用输入变量 `group_name` 进行赋值
- **description**：IAM用户组的描述，通过引用输入变量 `group_description` 进行赋值

### 5. 通过数据源查询IAM项目信息

在TF文件中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建IAM角色分配资源：

```hcl
variable "authorized_project_id" {
  description = "IAM项目的ID"
  type        = string
  default     = ""
  nullable    = false
}

variable "authorized_project_name" {
  description = "IAM项目的名称"
  type        = string
  default     = ""
  nullable    = true

  validation {
    condition     = !(var.authorized_project_name == "" && var.authorized_project_id == "")
    error_message = "当authorized_project_id未提供时，必须提供authorized_project_name。"
  }
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下符合条件的IAM项目信息，用于创建IAM角色分配资源
data "huaweicloud_identity_projects" "test" {
  count = var.authorized_project_id == "" ? 1 : 0

  name = var.authorized_project_name
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行IAM项目查询数据源，仅当 `var.authorized_project_id` 为空时创建数据源（即执行IAM项目查询）
- **name**：IAM项目的名称，通过引用输入变量 `authorized_project_name` 进行赋值

### 6. 创建IAM角色分配资源

在TF文件中添加以下脚本以告知Terraform创建IAM角色分配资源：

```hcl
variable "authorized_domain_id" {
  description = "IAM域的ID"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IAM角色分配资源
resource "huaweicloud_identity_role_assignment" "test" {
  group_id   = var.group_id != "" ? var.group_id : huaweicloud_identity_group.test[0].id
  role_id    = var.role_id != "" ? var.role_id : var.role_policy != "" ? huaweicloud_identity_role.test[0].id : try(data.huaweicloud_identity_role.test[0].id, "NOT_FOUND")
  domain_id  = var.authorized_domain_id != "" ? var.authorized_domain_id : null
  project_id = var.authorized_domain_id == "" ? var.authorized_project_id != "" ? var.authorized_project_id : try(data.huaweicloud_identity_projects.test[0].projects[0].id, "NOT_FOUND") : null
}
```

**参数说明**：
- **group_id**：IAM用户组的ID，如果指定了 `var.group_id` 则使用该值，否则引用前面创建的IAM用户组资源的ID
- **role_id**：IAM角色的ID，根据变量配置情况选择使用 `var.role_id`、创建的IAM角色资源ID或查询到的IAM角色数据源ID
- **domain_id**：IAM域的ID，如果指定了 `var.authorized_domain_id` 则使用该值，否则为null表示在项目级别授权
- **project_id**：IAM项目的ID，当 `var.authorized_domain_id` 为空时，如果指定了 `var.authorized_project_id` 则使用该值，否则使用查询到的IAM项目数据源ID；当 `var.authorized_domain_id` 不为空时，该参数为null表示在域级别授权

> 注意：IAM角色分配支持在域级别或项目级别进行授权。当指定 `domain_id` 时，表示在域级别授权；当指定 `project_id` 时，表示在项目级别授权。两者不能同时指定。

### 7. 创建随机密码资源

在TF文件中添加以下脚本以告知Terraform创建随机密码资源（当用户未指定密码时自动生成）：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建随机密码资源，用于为未指定密码的用户生成随机密码
resource "random_password" "test" {
  count = length([for v in var.users_configuration : true if v.password == "" || v.password == null]) > 0 ? 1 : 0

  length           = 16
  special          = true
  override_special = "_%@"
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建随机密码资源，仅当存在未指定密码的用户时创建资源
- **length**：密码的长度，设置为16个字符
- **special**：是否包含特殊字符，设置为true表示包含特殊字符
- **override_special**：特殊字符集，设置为"_%@"表示密码中可包含下划线、百分号和@符号

### 8. 创建IAM用户资源

在TF文件中添加以下脚本以告知Terraform创建IAM用户资源：

```hcl
variable "users_configuration" {
  description = "IAM用户的配置"
  type        = list(object({
    name     = string
    password = optional(string, "")
  }))
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IAM用户资源
resource "huaweicloud_identity_user" "test" {
  count = length(var.users_configuration)

  name     = lookup(var.users_configuration[count.index], "name", null)
  password = lookup(var.users_configuration[count.index], "password", null) != "" ? lookup(var.users_configuration[count.index], "password", null) : random_password.test[count.index].result
}
```

**参数说明**：
- **count**：资源的创建数，根据 `var.users_configuration` 列表的长度创建对应数量的IAM用户资源
- **name**：IAM用户的名称，从 `var.users_configuration` 列表中获取对应用户的name字段
- **password**：IAM用户的密码，如果用户配置中指定了密码则使用该密码，否则使用随机密码资源生成的密码

### 9. 创建IAM用户组成员关系资源

在TF文件中添加以下脚本以告知Terraform创建IAM用户组成员关系资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IAM用户组成员关系资源，将用户添加到用户组
resource "huaweicloud_identity_group_membership" "test" {
  group = var.group_id != "" ? var.group_id : huaweicloud_identity_group.test[0].id
  users = huaweicloud_identity_user.test[*].id
}
```

**参数说明**：
- **group**：IAM用户组的ID，如果指定了 `var.group_id` 则使用该值，否则引用前面创建的IAM用户组资源的ID
- **users**：IAM用户的ID列表，引用前面创建的所有IAM用户资源的ID，使用 `[*]` 语法获取所有用户资源的ID

### 10. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# IAM角色配置
role_name        = "tf_test_role"
role_policy      = <<EOT
{
  "Version": "1.1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "obs:*:*"
      ]
    },
    {
      "Effect": "Deny",
      "Action": [
        "obs:object:DeleteObjectVersion",
        "obs:object:DeleteAccessLabel",
        "obs:bucket:DeleteDirectColdAccessConfiguration",
        "obs:object:AbortMultipartUpload",
        "obs:bucket:DeleteBucketWebsite",
        "obs:object:DeleteObject",
        "obs:bucket:DeleteBucketPolicy",
        "obs:bucket:DeleteBucketCustomDomainConfiguration",
        "obs:object:RestoreObject",
        "obs:bucket:DeleteBucket",
        "obs:object:ModifyObjectMetaData",
        "obs:bucket:DeleteBucketInventoryConfiguration",
        "obs:bucket:DeleteReplicationConfiguration",
        "obs:bucket:DeleteBucketTagging"
      ]
    }
  ]
}
EOT
role_description = "Created by Terraform"

# IAM用户组配置
group_name = "tf_test_group"

# IAM项目配置
authorized_project_name = "cn-north-4"

# IAM用户配置
users_configuration = [
  {
    name = "tf_test_user"
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="role_name=my-role" -var="group_name=my-group"`
2. 环境变量：`export TF_VAR_role_name=my-role`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 11. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建IAM角色、用户组和用户，并通过用户组为用户授权
4. 运行 `terraform show` 查看已创建的IAM资源详情

## 参考信息

- [华为云IAM产品文档](https://support.huaweicloud.com/iam/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [IAM通过用户组授权的用户最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/iam/users-authorized-through-group)
