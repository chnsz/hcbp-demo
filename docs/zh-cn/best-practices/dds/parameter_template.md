# 部署参数模板

## 应用场景

文档数据库服务（DDS）支持通过参数模板统一管理实例的数据库参数，您可以将一组经过验证的参数配置保存为模板，并在创建或变更实例时快速应用，从而保证不同实例之间参数配置的一致性与可维护性。

本最佳实践将介绍如何使用Terraform自动化部署DDS参数模板，包括参数模板的创建及其名称、节点类型、数据库版本、参数映射和描述等配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DDS参数模板（huaweicloud_dds_parameter_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_parameter_template)

### 资源/数据源依赖关系

```
huaweicloud_dds_parameter_template
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DDS参数模板

在TF文件（如main.tf）中添加以下脚本以创建DDS参数模板：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DDS参数模板资源
variable "template_name" {
  description = "The name of the parameter template"
  type        = string
}

variable "template_mapping" {
  description = "The mapping between parameter names and parameter values"
  type        = map(string)
}

variable "template_node_type" {
  description = "The node type of the parameter template"
  type        = string
  default     = "mongos"
}

variable "database_version" {
  description = "The database version"
  type        = string
  default     = "4.0"
}

variable "template_description" {
  description = "The description of the parameter template"
  type        = string
  default     = ""
}

resource "huaweicloud_dds_parameter_template" "test" {
  name             = var.template_name
  parameter_values = var.template_mapping
  node_type        = var.template_node_type
  node_version     = var.database_version
  description      = var.template_description
}
```

**参数说明**：
- **name**：参数模板名称，通过引用输入变量 template_name 进行赋值
- **parameter_values**：参数名与参数值的映射关系，通过引用输入变量 template_mapping 进行赋值
- **node_type**：参数模板的节点类型，通过引用输入变量 template_node_type 进行赋值，可选值为 `mongos`、`shard`、`config`、`replica`、`readonly`、`shard_readonly` 或 `single`
- **node_version**：数据库版本，通过引用输入变量 database_version 进行赋值，可选值为 `5.0`、`4.4`、`4.2`、`4.0`、`3.4`
- **description**：参数模板描述，通过引用输入变量 template_description 进行赋值

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
template_name    = "tf_test_template"
template_mapping = {
  connPoolMaxConnsPerHost        = 500
  connPoolMaxShardedConnsPerHost = 500
}
template_description = "This is a parameter template created by terraform"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DDS参数模板
4. 运行 `terraform show` 查看已创建的DDS参数模板

## 参考信息

- [华为云文档数据库服务产品文档](https://support.huaweicloud.com/dds/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DDS参数模板最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/parameter-template)
