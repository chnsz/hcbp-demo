# 部署批量关联标签

## 应用场景

标签管理服务（Tag Management Service，TMS）是华为云提供的标签管理服务，支持为云资源添加、修改和删除标签，帮助您实现资源的分类管理和成本分析。通过批量关联标签功能，可以为多个云资源批量添加标签，提高标签管理的效率。批量关联标签支持为不同类型的资源（如ECS、VPC、RDS等）批量添加相同的标签，实现资源的统一分类和标识。本最佳实践将介绍如何使用Terraform自动化部署批量关联标签，包括查询项目列表和批量关联标签。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [项目列表查询数据源（data.huaweicloud_identity_projects）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### 资源

- [资源标签资源（huaweicloud_tms_resource_tags）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/tms_resource_tags)

### 资源/数据源依赖关系

```text
data.huaweicloud_identity_projects
    └── huaweicloud_tms_resource_tags
```

> 注意：资源标签需要指定项目ID，用于确定标签的作用范围。通过查询项目列表数据源可以获取当前区域下的项目信息。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询项目列表

在TF文件（如main.tf）中添加以下脚本以查询项目列表：

```hcl
variable "region_name" {
  description = "The region where the LTS service is located"
  type        = string
}

# 查询项目列表数据源
data "huaweicloud_identity_projects" "test" {
  name = var.region_name
}

# 通过本地值获取精确匹配的项目ID
locals {
  exact_project_id = try([for v in data.huaweicloud_identity_projects.test.projects : v.id if v.name == var.region_name][0], null)
}
```

**参数说明**：
- **name**：项目名称，通过引用输入变量 `region_name` 进行赋值，用于过滤项目
- **exact_project_id**：精确匹配的项目ID，通过本地值 `locals` 从项目列表中筛选出名称匹配的项目ID

> 注意：项目列表用于获取项目ID，后续批量关联标签时需要引用项目ID来确定标签的作用范围。

### 3. 批量关联标签

在TF文件（如main.tf）中添加以下脚本以批量关联标签：

```hcl
variable "associated_resources_configuration" {
  description = "The configuration of the associated resources"
  type        = list(object({
    type = string
    id   = string
  }))
}

variable "resource_tags" {
  description = "The tags of the resources"
  type        = map(string)
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建资源标签资源
resource "huaweicloud_tms_resource_tags" "test" {
  project_id = local.exact_project_id

  dynamic "resources" {
    for_each = var.associated_resources_configuration

    content {
      resource_type = resources.value.type
      resource_id   = resources.value.id
    }
  }

  tags = var.resource_tags
}
```

**参数说明**：
- **project_id**：项目ID，通过引用本地值 `exact_project_id` 进行赋值，用于确定标签的作用范围
- **resources**：关联资源列表，通过动态块 `dynamic "resources"` 根据输入变量 `associated_resources_configuration` 创建多个资源配置
  - **resource_type**：资源类型，通过引用输入变量中的 `type` 进行赋值，支持 `dcs`、`ecs`、`vpc` 等多种资源类型
  - **resource_id**：资源ID，通过引用输入变量中的 `id` 进行赋值
- **tags**：标签键值对，通过引用输入变量 `resource_tags` 进行赋值，格式为 `map(string)`

> 注意：批量关联标签可以为多个不同类型的资源批量添加相同的标签。资源类型和资源ID需要匹配，确保资源存在。标签键值对可以包含多个标签，用于资源的分类和标识。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 关联资源配置（必填）
associated_resources_configuration = [
  {
    type = "dcs"
    id   = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  }
]

# 资源标签配置（必填）
resource_tags = {
  foo   = "bar"
  owner = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="associated_resources_configuration=[{type=\"dcs\",id=\"xxx\"}]" -var="resource_tags={foo=\"bar\"}"`
2. 环境变量：`export TF_VAR_resource_tags='{"foo":"bar","owner":"terraform"}'`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始批量关联标签
4. 运行 `terraform show` 查看已关联的标签

## 参考信息

- [华为云TMS产品文档](https://support.huaweicloud.com/tms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [TMS批量关联标签最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/tms/batch-associate-tags)
