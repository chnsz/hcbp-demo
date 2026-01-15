# 部署密码策略

## 应用场景

统一身份认证服务（Identity and Access Management，IAM）是华为云提供的基础身份认证与访问管理服务，为华为云用户提供身份管理、权限管理和访问控制等核心功能。通过配置密码策略，可以设置密码的复杂度要求、有效期、重用规则等安全策略，提高账户安全性，满足企业级安全合规要求。本最佳实践将介绍如何使用Terraform自动化部署IAM密码策略，包括密码长度、字符组合、有效期、重用规则等安全策略的配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [IAM密码策略资源（huaweicloud_identityv5_password_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identityv5_password_policy)

### 资源/数据源依赖关系

```text
huaweicloud_identityv5_password_policy
```

> 注意：IAM密码策略资源是全局资源，用于配置IAM账户的密码安全策略。密码策略配置后，将应用于所有IAM用户。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建IAM密码策略资源

在TF文件（如main.tf）中添加以下脚本以创建IAM密码策略：

```hcl
variable "policy_max_consecutive_identical_chars" {
  description = "The maximum number of times that a character is allowed to consecutively present in a password"
  type        = number

  validation {
    condition     = var.policy_max_consecutive_identical_chars >= 0 && var.policy_max_consecutive_identical_chars <= 32
    error_message = "The valid value of the maximum number of times that a character is allowed to consecutively present in a password is range from 0 to 32"
  }
}

variable "policy_min_password_age" {
  description = "The minimum period (minutes) after which users are allowed to make a password change"
  type        = number

  validation {
    condition     = var.policy_min_password_age >= 0 && var.policy_min_password_age <= 1440
    error_message = "The valid value of the minimum period (minutes) after which users are allowed to make a password change is range from 0 to 1440"
  }
}

variable "policy_min_password_length" {
  description = "The minimum number of characters that a password must contain"
  type        = number

  validation {
    condition     = var.policy_min_password_length >= 8 && var.policy_min_password_length <= 32
    error_message = "The valid value of the minimum number of characters that a password must contain is range from 8 to 32"
  }
}

variable "policy_password_reuse_prevention" {
  description = "The password reuse prevention feature of the identity center's password policy indicates whether to prohibit the use of the same password as the previous one"
  type        = number
  default     = 3

  validation {
    condition     = var.policy_password_reuse_prevention >= 0 && var.policy_password_reuse_prevention <= 24
    error_message = "The valid value of the password reuse prevention feature of the identity center's password policy is range from 0 to 24"
  }
}

variable "policy_password_not_username_or_invert" {
  description = "Whether the password can be the username or the username spelled backwards"
  type        = bool
  default     = false
}

variable "policy_password_validity_period" {
  description = "The password validity period (days)"
  type        = number
  default     = 7

  validation {
    condition     = var.policy_password_validity_period >= 0 && var.policy_password_validity_period <= 180
    error_message = "The valid value of the password validity period (days) is range from 0 to 180"
  }
}

variable "policy_password_char_combination" {
  description = "The minimum number of character types that a password must contain"
  type        = number

  validation {
    condition     = var.policy_password_char_combination >= 2 && var.policy_password_char_combination <= 4
    error_message = "The valid value of the minimum number of character types that a password must contain is range from 2 to 4"
  }
}

variable "policy_allow_user_to_change_password" {
  description = "Whether IAM users are allowed to change their own passwords"
  type        = bool
  default     = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建IAM密码策略资源
resource "huaweicloud_identityv5_password_policy" "test" {
  maximum_consecutive_identical_chars = var.policy_max_consecutive_identical_chars
  minimum_password_age                = var.policy_min_password_age
  minimum_password_length             = var.policy_min_password_length
  password_reuse_prevention           = var.policy_password_reuse_prevention
  password_not_username_or_invert     = var.policy_password_not_username_or_invert
  password_validity_period            = var.policy_password_validity_period
  password_char_combination           = var.policy_password_char_combination
  allow_user_to_change_password       = var.policy_allow_user_to_change_password
}
```

**参数说明**：
- **maximum_consecutive_identical_chars**：密码中同一字符连续出现的最大次数，通过引用输入变量policy_max_consecutive_identical_chars进行赋值，取值范围为0-32
- **minimum_password_age**：密码最短使用时间（分钟），通过引用输入变量policy_min_password_age进行赋值，取值范围为0-1440
- **minimum_password_length**：密码最小长度，通过引用输入变量policy_min_password_length进行赋值，取值范围为8-32
- **password_reuse_prevention**：密码重用防护，通过引用输入变量policy_password_reuse_prevention进行赋值，取值范围为0-24，默认值为3
- **password_not_username_or_invert**：密码不能是用户名或用户名的倒序，通过引用输入变量policy_password_not_username_or_invert进行赋值，默认值为false
- **password_validity_period**：密码有效期（天），通过引用输入变量policy_password_validity_period进行赋值，取值范围为0-180，默认值为7
- **password_char_combination**：密码必须包含的字符类型最小数量，通过引用输入变量policy_password_char_combination进行赋值，取值范围为2-4
- **allow_user_to_change_password**：是否允许IAM用户修改自己的密码，通过引用输入变量policy_allow_user_to_change_password进行赋值，默认值为true

> 注意：IAM密码策略是全局资源，配置后将应用于所有IAM用户。密码策略参数都有取值范围限制，请根据实际安全需求合理配置。建议遵循最小权限原则，设置合理的密码复杂度要求，提高账户安全性。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# IAM密码策略配置
policy_max_consecutive_identical_chars = 2
policy_min_password_age                = 60
policy_min_password_length             = 8
policy_password_char_combination       = 2
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `policy_max_consecutive_identical_chars`可以设置密码中同一字符连续出现的最大次数，建议设置为2或更小
   - `policy_min_password_age`可以设置密码最短使用时间（分钟），建议设置为60分钟或更长
   - `policy_min_password_length`可以设置密码最小长度，建议设置为8或更长
   - `policy_password_reuse_prevention`可以设置密码重用防护，建议设置为3或更大
   - `policy_password_not_username_or_invert`可以设置为true，禁止密码是用户名或用户名的倒序
   - `policy_password_validity_period`可以设置密码有效期（天），建议设置为30-90天
   - `policy_password_char_combination`可以设置密码必须包含的字符类型最小数量，建议设置为2或更大
   - `policy_allow_user_to_change_password`可以设置为true，允许IAM用户修改自己的密码
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="policy_min_password_length=12" -var="policy_password_char_combination=3"`
2. 环境变量：`export TF_VAR_policy_min_password_length=12` 和 `export TF_VAR_policy_password_char_combination=3`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。IAM密码策略配置后，将应用于所有IAM用户，请根据实际安全需求合理配置密码策略参数。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建IAM密码策略：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建IAM密码策略
4. 运行 `terraform show` 查看已创建的IAM密码策略详情

> 注意：IAM密码策略是全局资源，配置后将应用于所有IAM用户。密码策略参数都有取值范围限制，请确保参数值在有效范围内。建议在配置密码策略前，先了解当前IAM用户的密码使用情况，避免过于严格的策略导致用户无法正常使用。

## 参考信息

- [华为云IAM产品文档](https://support.huaweicloud.com/iam/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [密码策略最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/iam/v5/password-policy)
