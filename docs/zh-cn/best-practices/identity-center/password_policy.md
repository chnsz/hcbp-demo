# 部署密码策略

## 应用场景

身份中心（Identity Center）是华为云提供的统一身份管理服务，支持跨云、跨应用的统一身份认证和授权管理。通过配置身份中心密码策略，可以设置密码的最大有效期、最小长度、重用防护、字符要求等安全策略，提高账户安全性，满足企业级安全合规要求。本最佳实践将介绍如何使用Terraform自动化部署身份中心密码策略，包括查询或创建身份中心实例、注册区域（可选）和配置密码策略。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [身份中心实例数据源（huaweicloud_identitycenter_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identitycenter_instance)

### 资源

- [身份中心注册区域资源（huaweicloud_identitycenter_registered_region）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_registered_region)
- [身份中心实例资源（huaweicloud_identitycenter_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_instance)
- [身份中心密码策略资源（huaweicloud_identitycenter_password_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_password_policy)

### 资源/数据源依赖关系

```text
huaweicloud_identitycenter_registered_region
    └── huaweicloud_identitycenter_instance
        └── huaweicloud_identitycenter_password_policy

data.huaweicloud_identitycenter_instance
    └── huaweicloud_identitycenter_password_policy
```

> 注意：身份中心密码策略资源需要依赖身份中心实例。如果实例不存在，需要先创建实例；如果实例已存在，可以通过数据源查询实例信息。创建实例时，如果需要在特定区域使用，可能需要先注册区域。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询身份中心实例数据源（可选）

在TF文件（如main.tf）中添加以下脚本以查询身份中心实例信息（当实例已存在时）：

```hcl
variable "is_instance_create" {
  description = "Whether to create the identity center instance"
  type        = bool
  default     = true
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下身份中心实例信息，用于配置密码策略
data "huaweicloud_identitycenter_instance" "test" {
  count = var.is_instance_create ? 0 : 1
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行身份中心实例查询数据源，仅当 `var.is_instance_create` 为false时创建数据源（即执行身份中心实例查询）

### 3. 创建身份中心注册区域资源（可选）

在TF文件（如main.tf）中添加以下脚本以注册区域（当需要创建实例且需要注册区域时）：

```hcl
variable "is_region_need_register" {
  description = "Whether to register the region"
  type        = bool
  default     = true
}

variable "region_name" {
  description = "The region where the LTS service is located"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建身份中心注册区域资源
resource "huaweicloud_identitycenter_registered_region" "test" {
  count = var.is_instance_create ? var.is_region_need_register ? 1 : 0 : 0

  region_id = var.region_name
}
```

**参数说明**：
- **count**：资源创建数量，用于控制是否创建注册区域资源，仅当需要创建实例且需要注册区域时创建
- **region_id**：区域ID，通过引用输入变量region_name进行赋值

### 4. 创建身份中心实例资源（可选）

在TF文件（如main.tf）中添加以下脚本以创建身份中心实例（当实例不存在时）：

```hcl
variable "instance_store_id_alias" {
  description = "The alias of the identity center instance"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建身份中心实例资源
resource "huaweicloud_identitycenter_instance" "test" {
  count = var.is_instance_create ? 1 : 0

  depends_on = [huaweicloud_identitycenter_registered_region.test]

  alias = var.instance_store_id_alias != "" ? var.instance_store_id_alias : null
}
```

**参数说明**：
- **count**：资源创建数量，用于控制是否创建身份中心实例资源，仅当 `var.is_instance_create` 为true时创建
- **depends_on**：显式依赖关系，确保在注册区域资源创建后再创建实例
- **alias**：实例别名，通过引用输入变量instance_store_id_alias进行赋值，可选参数，默认值为null

### 5. 创建身份中心密码策略资源

在TF文件（如main.tf）中添加以下脚本以创建身份中心密码策略：

```hcl
variable "policy_max_password_age" {
  description = "The max password age of the identity center password policy, unit in days"
  type        = number
  default     = 10

  validation {
    condition     = var.policy_max_password_age >= 1 && var.policy_max_password_age <= 1095
    error_message = "The valid value of the max password age is range from 1 to 1095"
  }
}

variable "policy_minimum_password_length" {
  description = "The minimum password length of the identity center password policy"
  type        = number
  default     = 10
}

variable "policy_password_reuse_prevention" {
  description = "The password reuse prevention feature of the identity center's password policy indicates whether to prohibit the use of the same password as the previous one"
  type        = bool
  default     = true
}

variable "policy_require_uppercase_characters" {
  description = "The require uppercase characters of the identity center password policy"
  type        = bool
  default     = true
}

variable "policy_require_lowercase_characters" {
  description = "The require lowercase characters of the identity center password policy"
  type        = bool
  default     = true
}

variable "policy_require_numbers" {
  description = "The require numbers of the identity center password policy"
  type        = bool
  default     = true
}

variable "policy_require_symbols" {
  description = "The require symbols of the identity center password policy"
  type        = bool
  default     = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建身份中心密码策略资源
resource "huaweicloud_identitycenter_password_policy" "test" {
  identity_store_id            = var.is_instance_create ? huaweicloud_identitycenter_instance.test[0].identity_store_id : data.huaweicloud_identitycenter_instance.test[0].identity_store_id
  max_password_age             = var.policy_max_password_age
  minimum_password_length      = var.policy_minimum_password_length
  password_reuse_prevention    = var.policy_password_reuse_prevention ? 1 : null
  require_uppercase_characters = var.policy_require_uppercase_characters
  require_lowercase_characters = var.policy_require_lowercase_characters
  require_numbers              = var.policy_require_numbers
  require_symbols              = var.policy_require_symbols
}
```

**参数说明**：
- **identity_store_id**：身份存储ID，通过引用身份中心实例资源或数据源进行赋值
- **max_password_age**：密码最大有效期（天），通过引用输入变量policy_max_password_age进行赋值，取值范围为1-1095，默认值为10
- **minimum_password_length**：密码最小长度，通过引用输入变量policy_minimum_password_length进行赋值，默认值为10
- **password_reuse_prevention**：密码重用防护，通过引用输入变量policy_password_reuse_prevention进行赋值，设置为1表示启用，设置为null表示禁用，默认值为1（启用）
- **require_uppercase_characters**：是否要求包含大写字母，通过引用输入变量policy_require_uppercase_characters进行赋值，默认值为true
- **require_lowercase_characters**：是否要求包含小写字母，通过引用输入变量policy_require_lowercase_characters进行赋值，默认值为true
- **require_numbers**：是否要求包含数字，通过引用输入变量policy_require_numbers进行赋值，默认值为true
- **require_symbols**：是否要求包含特殊字符，通过引用输入变量policy_require_symbols进行赋值，默认值为true

> 注意：身份中心密码策略需要依赖身份中心实例。如果实例不存在，需要先创建实例；如果实例已存在，可以通过数据源查询实例信息。密码策略参数需要根据实际安全需求合理配置，建议遵循最小权限原则，设置合理的密码复杂度要求，提高账户安全性。

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 身份中心密码策略配置
# 策略定义密码最小长度为8个字符，允许大写字母、小写字母、数字和特殊字符，密码有效期为30天
policy_max_password_age        = 30
policy_minimum_password_length = 8
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `is_instance_create`可以设置为true创建实例，或设置为false使用已存在的实例
   - `is_region_need_register`可以设置为true注册区域，或设置为false不注册区域
   - `instance_store_id_alias`可以设置实例别名，可选参数
   - `policy_max_password_age`可以设置密码最大有效期（天），建议设置为30-90天
   - `policy_minimum_password_length`可以设置密码最小长度，建议设置为8或更长
   - `policy_password_reuse_prevention`可以设置为true启用密码重用防护，或设置为false禁用
   - `policy_require_uppercase_characters`可以设置为true要求包含大写字母，或设置为false不要求
   - `policy_require_lowercase_characters`可以设置为true要求包含小写字母，或设置为false不要求
   - `policy_require_numbers`可以设置为true要求包含数字，或设置为false不要求
   - `policy_require_symbols`可以设置为true要求包含特殊字符，或设置为false不要求
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="policy_minimum_password_length=12" -var="policy_max_password_age=90"`
2. 环境变量：`export TF_VAR_policy_minimum_password_length=12` 和 `export TF_VAR_policy_max_password_age=90`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。身份中心密码策略需要依赖身份中心实例，请确保实例已存在或已配置创建实例。密码策略参数需要根据实际安全需求合理配置，建议遵循最小权限原则。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建身份中心密码策略：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建身份中心实例（如需要）和密码策略
4. 运行 `terraform show` 查看已创建的身份中心密码策略详情

> 注意：身份中心密码策略创建前需要确保身份中心实例已存在。如果实例不存在，需要先创建实例。创建实例时，如果需要在特定区域使用，可能需要先注册区域。建议在创建密码策略前，先确认身份中心实例的状态，并根据实际安全需求合理配置密码策略参数。

## 参考信息

- [华为云Identity Center产品文档](https://support.huaweicloud.com/identity-center/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [密码策略最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/identity-center/password-policy)
