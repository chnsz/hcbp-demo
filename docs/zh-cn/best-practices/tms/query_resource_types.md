# 部署资源类型查询

## 应用场景

标签管理服务（Tag Management Service，TMS）是华为云提供的标签管理服务，支持为云资源添加、修改和删除标签，帮助您实现资源的分类管理和成本分析。资源类型查询是TMS服务的重要功能，用于查询支持标签管理的云资源类型。通过查询资源类型，可以获取支持标签管理的服务名称和资源类型列表，为后续的标签管理操作提供基础信息。本最佳实践将介绍如何使用Terraform自动化部署资源类型查询，包括精确匹配和模糊匹配服务名称、模糊匹配资源类型名称等查询方式。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [资源类型查询数据源（data.huaweicloud_tms_resource_types）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/tms_resource_types)

### 资源/数据源依赖关系

```text
data.huaweicloud_tms_resource_types
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询资源类型

在TF文件（如main.tf）中添加以下脚本以查询资源类型：

```hcl
variable "exact_service_name" {
  description = "The exact service name to filter"
  type        = string
  default     = ""
}

variable "fuzzy_service_name" {
  description = "The fuzzy service name to filter"
  type        = string
  default     = ""
}

variable "fuzzy_resource_type_name" {
  description = "The fuzzy resource type name to filter"
  type        = string
  default     = ""
}

# 查询资源类型数据源
data "huaweicloud_tms_resource_types" "test" {
  service_name = var.exact_service_name != "" ? var.exact_service_name : null
}

# 通过本地值进行模糊匹配和过滤
locals {
  # 所有在TMS服务上注册的服务名称（基于用户指定的服务名称进行模糊匹配）
  regex_matched_service_names = distinct(var.fuzzy_service_name != "" ? [
    for v in data.huaweicloud_tms_resource_types.test.types[*].service_name : v if length(regexall(var.fuzzy_service_name, v)) > 0
  ] : data.huaweicloud_tms_resource_types.test.types[*].service_name)

  # 所有在TMS服务上注册的资源类型（包含资源类型名称和所属服务名称的对象）（基于用户指定的服务名称进行模糊匹配）
  regex_matched_resource_types_by_only_fuzzy_service_name = var.fuzzy_service_name != "" ? [
    for v in data.huaweicloud_tms_resource_types.test.types : v if length(regexall(var.fuzzy_service_name, v.service_name)) > 0
  ] : data.huaweicloud_tms_resource_types.test.types

  # 所有在TMS服务上注册的资源类型（包含资源类型名称和所属服务名称的对象）（基于用户指定的服务名称或资源类型名称进行模糊匹配）
  regex_matched_resource_types = var.fuzzy_resource_type_name != "" ? [
    for v in local.regex_matched_resource_types_by_only_fuzzy_service_name : v if length(regexall(var.fuzzy_resource_type_name, v.name)) > 0
  ] : local.regex_matched_resource_types_by_only_fuzzy_service_name
}
```

**参数说明**：
- **service_name**：服务名称，通过引用输入变量 `exact_service_name` 进行赋值，用于精确匹配服务名称，可选参数
- **regex_matched_service_names**：通过本地值 `locals` 进行模糊匹配的服务名称列表，当 `fuzzy_service_name` 不为空时进行模糊匹配，否则返回所有服务名称
- **regex_matched_resource_types_by_only_fuzzy_service_name**：通过本地值 `locals` 基于服务名称进行模糊匹配的资源类型列表，当 `fuzzy_service_name` 不为空时进行模糊匹配，否则返回所有资源类型
- **regex_matched_resource_types**：通过本地值 `locals` 基于服务名称或资源类型名称进行模糊匹配的资源类型列表，当 `fuzzy_resource_type_name` 不为空时进行模糊匹配，否则返回基于服务名称匹配的结果

> 注意：资源类型查询支持精确匹配和模糊匹配两种方式。精确匹配通过 `service_name` 参数指定服务名称，模糊匹配通过本地值使用正则表达式进行匹配。查询结果包含服务名称和资源类型名称，可以用于后续的标签管理操作。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 模糊匹配服务名称（可选）
fuzzy_service_name = "ccm"

# 模糊匹配资源类型名称（可选）
fuzzy_resource_type_name = "certificate"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="fuzzy_service_name=ccm" -var="fuzzy_resource_type_name=certificate"`
2. 环境变量：`export TF_VAR_fuzzy_service_name=ccm`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来查询资源类型：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看查询计划
3. 运行 `terraform apply` 执行查询操作
4. 运行 `terraform show` 查看查询结果，或使用 `terraform output` 输出本地值结果

## 参考信息

- [华为云TMS产品文档](https://support.huaweicloud.com/tms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [TMS资源类型查询最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/tms/query-resource-types)
