# 部署自定义扫描安全级别

## 应用场景

数据安全中心（Data Security Center，DSC）是华为云提供的一站式数据安全治理服务，支持对云上数据库、大数据、对象存储等多种数据源进行敏感数据扫描与分类分级。在敏感数据识别过程中，安全级别用于对扫描出的敏感数据进行分级标识，帮助用户按照数据敏感程度制定差异化的防护与治理策略。

本最佳实践将介绍如何使用Terraform自动化部署DSC自定义扫描安全级别，包括安全级别名称、控制台显示颜色编号和安全级别描述的配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DSC扫描安全级别（huaweicloud_dsc_scan_security_level）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_scan_security_level)

### 资源/数据源依赖关系

```
huaweicloud_dsc_scan_security_level
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DSC扫描安全级别

在TF文件（如main.tf）中添加以下脚本以创建DSC扫描安全级别：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DSC扫描安全级别资源
variable "security_level_name" {
  description = "The name of the security level"
  type        = string
}

variable "security_level_color_number" {
  description = "The color number of the security level displayed on the console"
  type        = number
  default     = 6
}

variable "security_level_description" {
  description = "The description of the security level"
  type        = string
  default     = ""
}

resource "huaweicloud_dsc_scan_security_level" "test" {
  security_level_name = var.security_level_name
  color_number        = var.security_level_color_number
  security_level_desc = var.security_level_description
}
```

**参数说明**：

- **security_level_name**：安全级别名称，通过引用输入变量 security_level_name 进行赋值，名称不能与当前DSC实例中已有的安全级别名称重复
- **color_number**：安全级别在控制台上显示的颜色编号，通过引用输入变量 security_level_color_number 进行赋值，默认值为6
- **security_level_desc**：安全级别描述，通过引用输入变量 security_level_description 进行赋值，默认值为空字符串

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 安全级别名称
security_level_name = "tfleveltest"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="security_level_name=tfleveltest"`
2. 环境变量：`export TF_VAR_security_level_name=tfleveltest`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DSC扫描安全级别
4. 运行 `terraform show` 查看已创建的DSC扫描安全级别

## 参考信息

- [华为云数据安全中心产品文档](https://support.huaweicloud.com/dsc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DSC自定义扫描安全级别最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dsc/custom-scan-security-level)
