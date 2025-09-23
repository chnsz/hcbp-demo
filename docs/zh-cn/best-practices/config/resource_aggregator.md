# 部署资源聚合器

## 应用场景

配置审计（Config）是华为云提供的一站式合规管理服务，帮助用户持续监控和评估云资源的配置合规性。Config服务提供预置的合规规则包和自定义规则，支持多种合规框架和标准，帮助企业建立完善的合规管理体系。

资源聚合器是Config服务的重要功能，用于跨账户或跨组织聚合云资源信息，实现统一的合规管理。通过资源聚合器，企业可以集中管理多个账户或组织中的资源，统一执行合规检查策略，提高合规管理的效率和一致性。资源聚合器支持账户级和组织级两种类型，为企业提供灵活的资源配置聚合方案。本最佳实践将介绍如何使用Terraform自动化部署Config资源聚合器，包括聚合器创建和账户配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [RMS资源聚合器资源（huaweicloud_rms_resource_aggregator）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rms_resource_aggregator)

### 资源/数据源依赖关系

```
huaweicloud_rms_resource_aggregator.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建资源聚合器

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建资源聚合器资源：

```hcl
variable "aggregator_name" {
  description = "资源聚合器名称"
  type        = string
}

variable "aggregator_type" {
  description = "资源聚合器类型，可以是ACCOUNT或ORGANIZATION"
  type        = string
}

variable "account_ids" {
  description = "源账户ID列表"
  type        = set(string)
  default     = []
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建资源聚合器资源
resource "huaweicloud_rms_resource_aggregator" "test" {
  name        = var.aggregator_name
  type        = var.aggregator_type
  account_ids = var.account_ids
}
```

**参数说明**：
- **name**：资源聚合器名称，通过引用输入变量aggregator_name进行赋值
- **type**：资源聚合器类型，通过引用输入变量aggregator_type进行赋值，支持ACCOUNT或ORGANIZATION
- **account_ids**：源账户ID列表，通过引用输入变量account_ids进行赋值

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 资源聚合器配置
aggregator_name = "tf_test_aggregator"
aggregator_type = "ACCOUNT"
account_ids     = ["account-id-1", "account-id-2", "account-id-3"]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="aggregator_name=my-aggregator" -var="aggregator_type=ACCOUNT"`
2. 环境变量：`export TF_VAR_aggregator_name=my-aggregator`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建资源聚合器
4. 运行 `terraform show` 查看已创建的资源聚合器

## 参考信息

- [华为云Config产品文档](https://support.huaweicloud.com/rms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Config最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rms)
