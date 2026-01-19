# 部署预设标签

## 应用场景

标签管理服务（Tag Management Service，TMS）是华为云提供的标签管理服务，支持为云资源添加、修改和删除标签，帮助您实现资源的分类管理和成本分析。预设标签是TMS服务的重要功能，用于创建可复用的标签模板，实现标签的标准化管理。通过创建预设标签，可以定义常用的标签键值对，后续创建资源时可以自动应用这些预设标签，提高标签管理的效率和一致性。本最佳实践将介绍如何使用Terraform自动化部署预设标签，包括预设标签的创建和配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [预设标签资源（huaweicloud_tms_tags）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/tms_tags)

### 资源/数据源依赖关系

```text
huaweicloud_tms_tags
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建预设标签

在TF文件（如main.tf）中添加以下脚本以创建预设标签：

```hcl
variable "preset_tags" {
  description = "The preset tags to be applied to the resource"
  type        = list(object({
    key   = string
    value = string
  }))
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建预设标签资源
resource "huaweicloud_tms_tags" "test" {
  dynamic "tags" {
    for_each = var.preset_tags

    content {
      key   = tags.value.key
      value = tags.value.value
    }
  }
}
```

**参数说明**：
- **tags**：标签列表，通过动态块 `dynamic "tags"` 根据输入变量 `preset_tags` 创建多个标签
  - **key**：标签键，通过引用输入变量中的 `key` 进行赋值
  - **value**：标签值，通过引用输入变量中的 `value` 进行赋值

> 注意：预设标签用于创建可复用的标签模板，支持创建多个标签键值对。创建预设标签后，可以在后续创建资源时自动应用这些标签，实现标签的标准化管理。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 预设标签配置（必填）
preset_tags = [
  {
    key   = "foo"
    value = "bar"
  },
  {
    key   = "owner"
    value = "terraform"
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="preset_tags=[{key=\"foo\",value=\"bar\"}]"`
2. 环境变量：`export TF_VAR_preset_tags='[{"key":"foo","value":"bar"}]'`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建预设标签
4. 运行 `terraform show` 查看已创建的预设标签

## 参考信息

- [华为云TMS产品文档](https://support.huaweicloud.com/tms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [TMS预设标签最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/tms/preset-tags)
