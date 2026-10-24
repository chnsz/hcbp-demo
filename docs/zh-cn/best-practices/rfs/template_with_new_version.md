# 部署模板及新版本

## 应用场景

资源编排服务（Resource Formation Service，RFS）是华为云提供的基础设施即代码（Infrastructure as Code，IaC）服务，用户可以通过模板描述云资源的组合与依赖关系，并由 RFS 统一完成资源的编排与交付。在实际使用中，模板往往需要随业务演进不断调整，因此 RFS 支持为同一模板发布多个版本，以便对模板内容进行迭代、追溯与回滚。

本最佳实践将介绍如何使用Terraform创建RFS模板，并为该模板发布一个包含扩展模板体的新版本。首先通过内联的 Terraform 模板体创建模板及其初始版本，然后基于扩展后的模板体为该模板发布新版本，从而演示模板版本化管理的完整流程。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [资源编排模板（huaweicloud_rfs_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_template)
- [资源编排模板版本（huaweicloud_rfs_template_version）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_template_version)

### 资源/数据源依赖关系

```
huaweicloud_rfs_template
    └── huaweicloud_rfs_template_version
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建资源编排模板

在TF文件（如main.tf）中添加以下脚本以创建RFS模板：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建资源编排模板
variable "template_name" {
  description = "The name of the RFS template"
  type        = string
}

variable "template_body" {
  description = "The Terraform template body of the RFS template"
  type        = string
}

variable "template_description" {
  description = "The description of the RFS template"
  type        = string
  default     = ""
}

variable "template_initial_version_description" {
  description = "The initial version description of the RFS template"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_template" "test" {
  template_name        = var.template_name
  template_body        = var.template_body
  template_description = var.template_description
  version_description  = var.template_initial_version_description
}
```

**参数说明**：
- **template_name**：模板名称，通过引用输入变量 template_name 进行赋值，仅允许字母、数字、中划线（-）和下划线（_），且在同一region内唯一
- **template_body**：模板体内容，通过引用输入变量 template_body 进行赋值，须为合法的 HCL 格式 Terraform 配置，本示例仅存储模板定义，不会创建模板体中的资源
- **template_description**：模板描述，通过引用输入变量 template_description 进行赋值
- **version_description**：模板初始版本描述，通过引用输入变量 template_initial_version_description 进行赋值

### 3. 为模板发布新版本

在TF文件（如main.tf）中添加以下脚本以为模板发布新版本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下为模板发布新版本
variable "template_version_body" {
  description = "The Terraform template body of the RFS template version"
  type        = string
}

variable "template_version_description" {
  description = "The description of the RFS template version"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_template_version" "test" {
  template_name       = huaweicloud_rfs_template.test.template_name
  template_id         = huaweicloud_rfs_template.test.template_id
  template_body       = var.template_version_body
  version_description = var.template_version_description
}
```

**参数说明**：
- **template_name**：模板名称，引用上一步创建的模板的 template_name 进行赋值
- **template_id**：模板ID，引用上一步创建的模板的 template_id 进行赋值
- **template_body**：新版本的模板体内容，通过引用输入变量 template_version_body 进行赋值，可在初始模板体基础上扩展资源定义
- **version_description**：新版本描述，通过引用输入变量 template_version_description 进行赋值

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
template_name = "tf-test-template"
template_body = <<EOF
resource "huaweicloud_vpc" "test" {
  name = "tf-test-vpc"
  cidr = "172.16.0.0/16"
}
EOF

template_version_body = <<EOF
resource "huaweicloud_vpc" "test" {
  name = "tf-test-vpc"
  cidr = "172.16.0.0/16"
}

resource "huaweicloud_vpc_subnet" "test" {
  name       = "tf-test-subnet"
  vpc_id     = huaweicloud_vpc.test.id
  cidr       = "172.16.1.0/24"
  gateway_ip = "172.16.1.1"
}
EOF
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

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建模板及其新版本
4. 运行 `terraform show` 查看已创建的模板及其新版本

## 参考信息

- [华为云资源编排服务（RFS）产品文档](https://support.huaweicloud.com/rfs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RFS模板及新版本最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rfs/template-with-new-version)
