# 部署合规规则包

## 应用场景

配置审计（Config）是华为云提供的一站式合规管理服务，帮助用户持续监控和评估云资源的配置合规性。Config服务提供预置的合规规则包和自定义规则，支持多种合规框架和标准，帮助企业建立完善的合规管理体系。

合规规则包是Config服务的核心功能，用于定义和管理一组相关的合规规则。通过合规规则包，企业可以快速部署符合特定合规框架或标准的规则集合，实现自动化的合规检查和评估。合规规则包支持预置模板和自定义配置，提供灵活的规则管理能力，为企业提供全面的合规管理解决方案。本最佳实践将介绍如何使用Terraform自动化部署Config合规规则包，包括规则包模板查询和合规规则包创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [RMS规则包模板查询数据源（data.huaweicloud_rms_assignment_package_templates）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rms_assignment_package_templates)

### 资源

- [RMS规则包资源（huaweicloud_rms_assignment_package）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rms_assignment_package)

### 资源/数据源依赖关系

```
data.huaweicloud_rms_assignment_package_templates.test
    └── huaweicloud_rms_assignment_package.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询规则包模板

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建合规规则包：

```hcl
variable "template_key" {
  description = "内置规则包模板名称"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的规则包模板信息，用于创建合规规则包
data "huaweicloud_rms_assignment_package_templates" "test" {
  template_key = var.template_key
}
```

**参数说明**：
- **template_key**：内置规则包模板名称，通过引用输入变量template_key进行赋值

### 3. 创建合规规则包

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建合规规则包资源：

```hcl
variable "assignment_package_name" {
  description = "规则包名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建合规规则包资源
resource "huaweicloud_rms_assignment_package" "test" {
  name         = var.assignment_package_name
  template_key = data.huaweicloud_rms_assignment_package_templates.test.templates.0.template_key

  dynamic "vars_structure" {
    for_each = data.huaweicloud_rms_assignment_package_templates.test.templates.0.parameters

    content {
      var_key   = vars_structure.value["name"]
      var_value = vars_structure.value["default_value"]
    }
  }
}
```

**参数说明**：
- **name**：规则包名称，通过引用输入变量assignment_package_name进行赋值
- **template_key**：模板键名，通过引用规则包模板数据源的第一个模板键名进行赋值
- **vars_structure**：变量结构，动态创建规则包参数配置
  - **var_key**：变量键名，使用模板参数的名称
  - **var_value**：变量值，使用模板参数的默认值

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 规则包模板配置
template_key = "Operational-Best-Practices-for-ECS.tf.json"

# 合规规则包配置
assignment_package_name = "tf_test_assignment_package"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="assignment_package_name=my-package" -var="template_key=my-template"`
2. 环境变量：`export TF_VAR_assignment_package_name=my-package`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建合规规则包
4. 运行 `terraform show` 查看已创建的合规规则包

## 参考信息

- [华为云Config产品文档](https://support.huaweicloud.com/rms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Config最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rms)
