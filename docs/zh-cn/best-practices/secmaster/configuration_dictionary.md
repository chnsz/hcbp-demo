# 部署配置字典

## 应用场景

安全云脑（SecMaster）是华为云原生的新一代安全运营中心，提供云上资产管理、安全态势管理、安全信息和事件管理、安全编排与自动响应等能力。配置字典用于维护告警、事件等业务对象中可选的枚举值集合，例如告警评论状态、处置结论等，使安全运营流程中的字段取值保持统一和规范。

本最佳实践将介绍如何使用Terraform自动化部署一个安全云脑配置字典，包括字典基本信息、语言环境、版本号、父级字典以及扩展字段等配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [安全云脑配置字典（huaweicloud_secmaster_configuration_dictionary）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_configuration_dictionary)

### 资源/数据源依赖关系

```
huaweicloud_secmaster_configuration_dictionary
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建安全云脑配置字典

在TF文件（如main.tf）中添加以下脚本以创建安全云脑配置字典：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全云脑配置字典资源
variable "dict_id" {
  description = "The dictionary ID"
  type        = string
}

variable "dict_key" {
  description = "The dictionary key"
  type        = string
}

variable "dict_code" {
  description = "The dictionary code"
  type        = string
}

variable "dict_val" {
  description = "The dictionary value"
  type        = string
}

variable "language" {
  description = "The language environment. Valid values: zh, en"
  type        = string
  default     = "zh"
}

variable "dict_version" {
  description = "The version number"
  type        = string
  default     = "1.0.0"
}

variable "dict_pkey" {
  description = "The parent key of the dictionary"
  type        = string
  default     = ""
}

variable "dict_pcode" {
  description = "The parent code of the dictionary"
  type        = string
  default     = ""
}

variable "scope" {
  description = "The domain to which the dictionary belongs"
  type        = string
  default     = "ALERT"
}

variable "description" {
  description = "The description of the dictionary"
  type        = string
  default     = ""
}

variable "extend_field" {
  description = "The extension field of the dictionary"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_secmaster_configuration_dictionary" "test" {
  dict_id      = var.dict_id
  dict_key     = var.dict_key
  dict_code    = var.dict_code
  dict_val     = var.dict_val
  language     = var.language
  version      = var.dict_version
  dict_pkey    = var.dict_pkey
  dict_pcode   = var.dict_pcode
  scope        = var.scope
  description  = var.description
  extend_field = var.extend_field
  is_built_in  = false

  lifecycle {
    ignore_changes = [
      is_built_in,
    ]
  }
}
```

**参数说明**：
- **dict_id**：字典ID，通过引用输入变量 dict_id 进行赋值，创建后不可修改
- **dict_key**：字典键，通过引用输入变量 dict_key 进行赋值，创建后不可修改
- **dict_code**：字典编码，通过引用输入变量 dict_code 进行赋值
- **dict_val**：字典值，通过引用输入变量 dict_val 进行赋值
- **language**：语言环境，通过引用输入变量 language 进行赋值，有效值为 zh、en，创建后不可修改
- **version**：版本号，通过引用输入变量 dict_version 进行赋值，创建后不可修改
- **dict_pkey**：父级字典键，通过引用输入变量 dict_pkey 进行赋值
- **dict_pcode**：父级字典编码，通过引用输入变量 dict_pcode 进行赋值
- **scope**：字典所属域，通过引用输入变量 scope 进行赋值，创建后不可修改
- **description**：字典描述，通过引用输入变量 description 进行赋值
- **extend_field**：字典扩展字段，通过引用输入变量 extend_field 进行赋值
- **is_built_in**：是否为内置字典，此处设置为 false 以创建用户自定义字典；该属性在导入后可能产生漂移，因此通过 lifecycle 的 ignore_changes 忽略其变更

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 配置字典变量
dict_id   = "3027"
dict_key  = "alert_comments"
dict_code = "Open"
dict_val  = "Open"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="dict_id=3027"`
2. 环境变量：`export TF_VAR_dict_id=3027`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建安全云脑配置字典
4. 运行 `terraform show` 查看已创建的安全云脑配置字典

## 参考信息

- [华为云安全云脑产品文档](https://support.huaweicloud.com/secmaster/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [安全云脑配置字典最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/configuration-dictionary)
