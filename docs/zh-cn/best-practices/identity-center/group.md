# 部署用户组

## 应用场景

身份中心（Identity Center）是华为云提供的统一身份管理服务，支持跨云、跨应用的统一身份认证和授权管理。通过创建用户组，可以将多个用户进行分组管理，统一配置权限策略、执行访问控制和进行身份管理，提高身份管理的效率和便捷性。本最佳实践将介绍如何使用Terraform自动化部署身份中心用户组，包括查询或创建身份中心实例、注册区域（可选）和创建用户组。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [身份中心实例数据源（huaweicloud_identitycenter_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identitycenter_instance)

### 资源

- [身份中心注册区域资源（huaweicloud_identitycenter_registered_region）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_registered_region)
- [身份中心实例资源（huaweicloud_identitycenter_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_instance)
- [身份中心用户组资源（huaweicloud_identitycenter_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identitycenter_group)

### 资源/数据源依赖关系

```text
huaweicloud_identitycenter_registered_region
    └── huaweicloud_identitycenter_instance
        └── huaweicloud_identitycenter_group

data.huaweicloud_identitycenter_instance
    └── huaweicloud_identitycenter_group
```

> 注意：身份中心用户组资源需要依赖身份中心实例。如果实例不存在，需要先创建实例；如果实例已存在，可以通过数据源查询实例信息。创建实例时，如果需要在特定区域使用，可能需要先注册区域。

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

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下身份中心实例信息，用于创建用户组
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

### 5. 创建身份中心用户组资源

在TF文件（如main.tf）中添加以下脚本以创建身份中心用户组：

```hcl
variable "group_name" {
  description = "The name of the identity center group"
  type        = string
}

variable "group_description" {
  description = "The description of the identity center group"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建身份中心用户组资源
resource "huaweicloud_identitycenter_group" "test" {
  identity_store_id = var.is_instance_create ? huaweicloud_identitycenter_instance.test[0].identity_store_id : data.huaweicloud_identitycenter_instance.test[0].identity_store_id
  name              = var.group_name
  description       = var.group_description
}
```

**参数说明**：
- **identity_store_id**：身份存储ID，通过引用身份中心实例资源或数据源进行赋值
- **name**：用户组名称，通过引用输入变量group_name进行赋值
- **description**：用户组描述，通过引用输入变量group_description进行赋值，可选参数，默认值为空字符串

> 注意：身份中心用户组需要依赖身份中心实例。如果实例不存在，需要先创建实例；如果实例已存在，可以通过数据源查询实例信息。创建实例时，如果需要在特定区域使用，可能需要先注册区域。

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 身份中心用户组配置
group_name = "tf_test_group"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `is_instance_create`可以设置为true创建实例，或设置为false使用已存在的实例
   - `is_region_need_register`可以设置为true注册区域，或设置为false不注册区域
   - `instance_store_id_alias`可以设置实例别名，可选参数
   - `group_name`需要设置用户组名称
   - `group_description`可以设置用户组描述信息，可选参数
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="group_name=my-group" -var="is_instance_create=true"`
2. 环境变量：`export TF_VAR_group_name=my-group` 和 `export TF_VAR_is_instance_create=true`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。身份中心用户组需要依赖身份中心实例，请确保实例已存在或已配置创建实例。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建身份中心用户组：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建身份中心实例（如需要）和用户组
4. 运行 `terraform show` 查看已创建的身份中心用户组详情

> 注意：身份中心用户组创建前需要确保身份中心实例已存在。如果实例不存在，需要先创建实例。创建实例时，如果需要在特定区域使用，可能需要先注册区域。建议在创建用户组前，先确认身份中心实例的状态。

## 参考信息

- [华为云Identity Center产品文档](https://support.huaweicloud.com/identity-center/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [用户组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/identity-center/group)
