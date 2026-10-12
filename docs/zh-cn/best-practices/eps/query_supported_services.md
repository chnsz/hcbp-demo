# 部署企业项目支持服务查询

## 应用场景

企业项目管理服务（Enterprise Project Management Service，EPS）是华为云提供的企业级资源管理服务，支持企业以项目为维度对云上资源进行分组、授权和成本核算。在使用企业项目对资源进行归集与授权前，通常需要先了解当前区域下 EPS 支持哪些云服务以及这些服务对应的资源类型，以便规划企业项目与资源的归属关系。

本最佳实践将介绍如何使用Terraform查询EPS支持的服务列表，通过数据源按显示语言、服务名称等条件进行过滤，帮助您快速获取可用于企业项目管理的服务清单，为后续的资源分组、授权与成本核算提供依据。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [企业项目支持服务列表（data.huaweicloud_enterprise_project_services）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/enterprise_project_services)

### 资源/数据源依赖关系

```
data.huaweicloud_enterprise_project_services
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询企业项目支持的服务列表

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询企业项目支持的服务列表
variable "exact_locale" {
  description = "The display language for service and resource type names"
  type        = string
  default     = ""
  nullable    = false
}

variable "fuzzy_service_name" {
  description = "The fuzzy service name to filter"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_enterprise_project_services" "test" {
  locale  = var.exact_locale != "" ? var.exact_locale : null
  service = var.fuzzy_service_name != "" ? var.fuzzy_service_name : null
}
```

**参数说明**：

- **locale**：服务名称与资源类型名称的显示语言，通过引用输入变量 exact_locale 进行赋值，当变量为空字符串时该参数不生效
- **service**：用于模糊匹配的服务名称，通过引用输入变量 fuzzy_service_name 进行赋值，当变量为空字符串时该参数不生效

### 3. 按精确服务名称过滤查询结果

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下按精确服务名称过滤查询结果
variable "exact_service_name" {
  description = "The exact service name to filter"
  type        = string
  default     = ""
  nullable    = false
}

locals {
  matched_services = data.huaweicloud_enterprise_project_services.test.services

  # 按用户指定的精确服务名称过滤 EPS 支持的服务列表
  matched_service_by_only_exact_service_name = var.exact_service_name != "" ? [
    for v in data.huaweicloud_enterprise_project_services.test.services : v if v.service == var.exact_service_name
  ] : data.huaweicloud_enterprise_project_services.test.services
}
```

**参数说明**：

- **matched_services**：数据源返回的全部服务列表，通过引用数据源 huaweicloud_enterprise_project_services.test 的 services 属性进行赋值
- **matched_service_by_only_exact_service_name**：按精确服务名称过滤后的服务列表，通过引用输入变量 exact_service_name 进行赋值，当变量为空字符串时返回全部服务列表

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "<YOUR_ACCESS_KEY>"
secret_key  = "<YOUR_SECRET_KEY>"

# 过滤变量
exact_locale       = "en-us"
fuzzy_service_name = "er"
exact_service_name = "ServiceStage"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="fuzzy_service_name=er"`
2. 环境变量：`export TF_VAR_fuzzy_service_name=er`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来查询企业项目支持的服务列表：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始查询企业项目支持的服务列表
4. 运行 `terraform show` 查看已查询到的企业项目支持的服务列表

## 参考信息

- [华为云企业项目管理服务产品文档](https://support.huaweicloud.com/eps/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [企业项目支持服务查询最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eps/query-supported-services)
