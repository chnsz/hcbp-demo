# 部署企业项目

## 应用场景

企业项目管理服务（Enterprise Project Management Service，EPS）是华为云提供的企业级资源管理服务，支持企业以项目为单位对云资源进行统一规划、分类和授权，实现资源的分组管理与成本归集。通过企业项目，企业可以将不同业务、部门或环境的资源划分到独立的项目下，并结合统一身份认证服务（IAM）实现精细化的权限控制。

本最佳实践将介绍如何使用Terraform自动化部署企业项目，包括企业项目的创建及其名称、描述、类型、启用状态等核心参数的配置，帮助您快速上手企业项目的自动化管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [企业项目（huaweicloud_enterprise_project）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/enterprise_project)

### 资源/数据源依赖关系

```
huaweicloud_enterprise_project
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建企业项目

在TF文件（如main.tf）中添加以下脚本以创建企业项目：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建企业项目资源
variable "enterprise_project_name" {
  description = "The name of the enterprise project"
  type        = string
}

variable "enterprise_project_description" {
  description = "The description of the enterprise project"
  type        = string
  default     = ""
}

variable "enterprise_project_type" {
  description = "The type of the enterprise project. Valid values are poc and prod"
  type        = string
  default     = "prod"
}

variable "enterprise_project_enable" {
  description = "Whether to enable the enterprise project"
  type        = bool
  default     = true
}

variable "skip_disable_on_destroy" {
  description = "Whether to skip disabling the enterprise project on destroy"
  type        = bool
  default     = false
}

variable "delete_flag" {
  description = "Whether to delete the enterprise project on destroy"
  type        = bool
  default     = true
}

resource "huaweicloud_enterprise_project" "test" {
  name                    = var.enterprise_project_name
  description             = var.enterprise_project_description
  type                    = var.enterprise_project_type
  enable                  = var.enterprise_project_enable
  skip_disable_on_destroy = var.skip_disable_on_destroy
  delete_flag             = var.delete_flag
}
```

**参数说明**：

- **name**：企业项目名称，通过引用输入变量 enterprise_project_name 进行赋值，名称在账号内必须唯一，且不能包含任何形式的“default”字样
- **description**：企业项目描述，通过引用输入变量 enterprise_project_description 进行赋值
- **type**：企业项目类型，通过引用输入变量 enterprise_project_type 进行赋值，取值范围为 poc 和 prod
- **enable**：是否启用企业项目，通过引用输入变量 enterprise_project_enable 进行赋值
- **skip_disable_on_destroy**：销毁时是否跳过禁用企业项目操作，通过引用输入变量 skip_disable_on_destroy 进行赋值
- **delete_flag**：销毁时是否删除企业项目，通过引用输入变量 delete_flag 进行赋值，启用该选项前请确保项目下没有关联资源

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 企业项目配置
enterprise_project_name        = "tf-test-eps"
enterprise_project_description = "Terraform EPS basic example"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="enterprise_project_name=my-eps"`
2. 环境变量：`export TF_VAR_enterprise_project_name=my-eps`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建企业项目
4. 运行 `terraform show` 查看已创建的企业项目

## 参考信息

- [华为云企业项目管理服务产品文档](https://support.huaweicloud.com/eps/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EPS企业项目最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eps/basic)
