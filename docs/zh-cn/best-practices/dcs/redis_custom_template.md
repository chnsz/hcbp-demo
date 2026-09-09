# 部署Redis自定义模板

## 应用场景

分布式缓存服务（DCS）提供系统预置模板和用户自定义模板，用于快速创建符合业务需求的Redis实例。当系统模板的参数配置无法满足特定业务要求时，您可以通过自定义模板基于已有模板进行参数覆盖和调整，实现参数配置的复用与标准化。

本最佳实践将介绍如何使用Terraform基于系统模板或用户模板创建DCS Redis自定义模板，并覆盖部分参数配置，帮助您通过基础设施即代码（IaC）的方式高效管理Redis参数模板。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DCS自定义模板（huaweicloud_dcs_custom_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_custom_template)

### 资源/数据源依赖关系

```
huaweicloud_dcs_custom_template
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DCS Redis自定义模板

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DCS Redis自定义模板
variable "source_template_id" {
  description = "The ID of the source template to create the custom template from"
  type        = string
}

variable "source_type" {
  description = "The type of the source template. Valid values: sys, user"
  type        = string
  default     = "sys"
}

variable "template_name" {
  description = "The name of the custom template"
  type        = string
}

variable "template_description" {
  description = "The description of the custom template"
  type        = string
  default     = ""
}

variable "template_params" {
  description = "The template params to override, mapping param names to values"
  type        = map(string)
  default     = {
    "timeout" = "200"
  }
}

resource "huaweicloud_dcs_custom_template" "test" {
  template_id = var.source_template_id
  source_type = var.source_type
  name        = var.template_name
  description = var.template_description

  dynamic "params" {
    for_each = var.template_params

    content {
      param_name  = params.key
      param_value = params.value
    }
  }
}
```

**参数说明**：
- **template_id**：通过引用输入变量 source_template_id 进行赋值，指定源模板的ID。
- **source_type**：通过引用输入变量 source_type 进行赋值，指定源模板的类型，支持sys（系统模板）和user（用户模板）。
- **name**：通过引用输入变量 template_name 进行赋值，指定自定义模板的名称。
- **description**：通过引用输入变量 template_description 进行赋值，指定自定义模板的描述信息。
- **params**：通过引用输入变量 template_params 进行赋值，以键值对形式覆盖模板参数，其中param_name为参数名，param_value为参数值。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
source_template_id   = "16"
source_type          = "sys"
template_name        = "tf_test_custom_template"
template_description = "terraform test custom template"
template_params      = {
  "timeout"          = "200"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="template_name=my-template"`
2. 环境变量：`export TF_VAR_template_name=my-template`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DCS Redis自定义模板
4. 运行 `terraform show` 查看已创建的DCS Redis自定义模板

## 参考信息

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DCS Redis自定义模板最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-custom-template)
