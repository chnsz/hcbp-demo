# 部署模板

## 应用场景

资源治理中心（Resource Governance Center，RGC）是华为云提供的资源治理服务，支持多账号管理、组织单元管理、蓝图配置等功能，帮助您统一管理和治理云上资源。通过创建RGC模板，可以定义资源的部署蓝图，实现资源的自动化部署和管理。模板可以是预定义模板或自定义模板，预定义模板由华为云提供，自定义模板可以根据实际需求进行配置。本最佳实践将介绍如何使用Terraform自动化部署RGC模板，包括预定义模板和自定义模板的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [模板资源（huaweicloud_rgc_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rgc_template)

### 资源/数据源依赖关系

```text
huaweicloud_rgc_template
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建模板

在TF文件（如main.tf）中添加以下脚本以创建RGC模板：

```hcl
variable "template_name" {
  description = "The name of the template"
  type        = string
}

variable "template_type" {
  description = "The type of the template"
  type        = string
  default     = "predefined"
}

variable "template_description" {
  description = "The description of the customized template"
  type        = string
  default     = null
}

variable "template_body" {
  description = "The content of the customized template"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建模板资源
resource "huaweicloud_rgc_template" "test" {
  template_name        = var.template_name
  template_type        = var.template_type
  template_description = var.template_description
  template_body        = var.template_body
}
```

**参数说明**：
- **template_name**：模板名称，通过引用输入变量 `template_name` 进行赋值
- **template_type**：模板类型，通过引用输入变量 `template_type` 进行赋值，可选值包括 `predefined`（预定义模板）和 `customized`（自定义模板），默认为 `predefined`
- **template_description**：自定义模板的描述，通过引用输入变量 `template_description` 进行赋值，仅当 `template_type` 为 `customized` 时有效，可选参数
- **template_body**：自定义模板的内容，通过引用输入变量 `template_body` 进行赋值，仅当 `template_type` 为 `customized` 时有效，可选参数

> 注意：预定义模板由华为云提供，创建时只需指定模板名称和类型。自定义模板需要提供模板描述和模板内容，模板内容通常为JSON格式的蓝图配置。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 模板基本信息（必填）
template_name = "tf_test_template"
template_type = "predefined"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

如果创建自定义模板，可以在`terraform.tfvars`文件中添加以下内容：

```hcl
# 自定义模板配置
template_name        = "tf_test_customized_template"
template_type        = "customized"
template_description = "This is a customized template for resource deployment"
template_body        = jsonencode({
  version = "1.0"
  resources = {
    # 模板内容配置
  }
})
```

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="template_name=tf_test_template" -var="template_type=predefined"`
2. 环境变量：`export TF_VAR_template_name=tf_test_template`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建模板
4. 运行 `terraform show` 查看已创建的模板

## 参考信息

- [华为云RGC产品文档](https://support.huaweicloud.com/rgc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RGC模板最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rgc/template)
