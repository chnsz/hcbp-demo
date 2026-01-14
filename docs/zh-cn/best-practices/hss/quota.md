# 部署配额

## 应用场景

主机安全服务（Host Security Service，HSS）是华为云提供的主机安全防护服务，提供资产管理、漏洞管理、入侵检测、基线检查等功能，帮助您全面保护云上主机的安全。通过购买HSS配额，可以为云上主机提供包年包月的安全防护服务，包括实时监控、威胁检测、漏洞扫描、基线检查等功能，保障主机安全。本最佳实践将介绍如何使用Terraform自动化部署HSS配额，购买包年包月的HSS防护配额，为云上主机提供持续的安全防护能力。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [HSS配额资源（huaweicloud_hss_quota）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/hss_quota)

### 资源/数据源依赖关系

```text
huaweicloud_hss_quota
```

> 注意：HSS配额资源是独立的资源，用于购买HSS防护配额。购买配额后，可以将配额分配给需要防护的主机使用。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建HSS配额资源

在TF文件（如main.tf）中添加以下脚本以创建HSS配额：

```hcl
variable "quota_version" {
  description = "The protection quota version"
  type        = string
}

variable "period_unit" {
  description = "The charging period unit of the quota"
  type        = string
}

variable "period" {
  description = "The charging period of the quota"
  type        = number
}

variable "is_auto_renew" {
  description = "Whether auto-renew is enabled"
  type        = bool
  default     = false
}

variable "enterprise_project_id" {
  description = "The enterprise project ID to which the HSS quota belongs"
  type        = string
  default     = null
}

variable "quota_tags" {
  description = "The key/value pairs to associate with the HSS quota"
  type        = map(string)
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建HSS配额资源
resource "huaweicloud_hss_quota" "test" {
  version               = var.quota_version
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.is_auto_renew
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.quota_tags
}
```

**参数说明**：
- **version**：防护配额版本，通过引用输入变量quota_version进行赋值，例如"hss.version.enterprise"表示企业版
- **period_unit**：计费周期单位，通过引用输入变量period_unit进行赋值，可选值为"month"（月）或"year"（年）
- **period**：计费周期，通过引用输入变量period进行赋值，表示购买的时长，例如1表示1个月或1年
- **auto_renew**：是否自动续费，通过引用输入变量is_auto_renew进行赋值，默认值为false
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，可选参数，默认值为null
- **tags**：配额标签，通过引用输入变量quota_tags进行赋值，可选参数，默认值为null

> 注意：HSS配额采用包年包月计费模式，需要指定计费周期单位和周期。购买配额后，可以将配额分配给需要防护的主机使用。配额版本需要根据实际需求选择，常见版本包括基础版、企业版等。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# HSS配额配置
quota_version         = "hss.version.enterprise"
period_unit           = "month"
period                = 1
is_auto_renew         = false
enterprise_project_id = "0"
quota_tags            = {
  foo = "bar"
  key = "value"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `quota_version`可以设置为"hss.version.enterprise"（企业版）或其他支持的配额版本
   - `period_unit`可以设置为"month"（月）或"year"（年）
   - `period`可以设置为购买的时长，例如1表示1个月或1年
   - `is_auto_renew`可以设置为true，表示启用自动续费
   - `enterprise_project_id`可以设置企业项目ID，如果不需要可以省略或设置为"0"
   - `quota_tags`可以设置配额标签，用于资源分类和管理，如果不需要可以省略
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="quota_version=hss.version.enterprise" -var="period_unit=month" -var="period=1"`
2. 环境变量：`export TF_VAR_quota_version=hss.version.enterprise` 和 `export TF_VAR_period_unit=month`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。HSS配额采用包年包月计费模式，购买后会产生费用，请根据实际需求选择合适的配额版本和购买时长。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建HSS配额：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建HSS配额
4. 运行 `terraform show` 查看已创建的HSS配额详情

> 注意：HSS配额创建后会产生费用，请确保账户余额充足。购买配额后，可以将配额分配给需要防护的主机使用。如果启用了自动续费，配额到期前会自动续费。

## 参考信息

- [华为云HSS产品文档](https://support.huaweicloud.com/hss/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [配额最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/hss/prepaid-quota)
