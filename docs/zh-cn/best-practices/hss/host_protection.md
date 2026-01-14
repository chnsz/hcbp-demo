# 部署主机防护

## 应用场景

主机安全服务（Host Security Service，HSS）是华为云提供的主机安全防护服务，提供资产管理、漏洞管理、入侵检测、基线检查等功能，帮助您全面保护云上主机的安全。通过为已存在的主机启用HSS防护，可以为云上主机提供全面的安全防护能力，包括实时监控、威胁检测、漏洞扫描、基线检查等功能，保障主机安全。本最佳实践将介绍如何使用Terraform自动化部署HSS主机防护，为已存在的主机启用按需计费的HSS防护服务。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [HSS主机防护资源（huaweicloud_hss_host_protection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/hss_host_protection)

### 资源/数据源依赖关系

```text
huaweicloud_hss_host_protection
```

> 注意：HSS主机防护资源需要依赖已存在的主机（ECS实例），该主机需要已安装HSS agent。主机ID通过输入变量进行指定。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建HSS主机防护资源

在TF文件（如main.tf）中添加以下脚本以创建HSS主机防护：

```hcl
variable "host_id" {
  description = "The host ID for the host protection"
  type        = string
}

variable "protection_version" {
  description = "The protection version enabled by the host"
  type        = string
}

variable "is_wait_host_available" {
  description = "Whether to wait for the host agent status to become online"
  type        = bool
  default     = false
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the host protection belongs"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建HSS主机防护资源
resource "huaweicloud_hss_host_protection" "test" {
  host_id                = var.host_id
  version                = var.protection_version
  charging_mode          = "postPaid"
  is_wait_host_available = var.is_wait_host_available
  enterprise_project_id  = var.enterprise_project_id
}
```

**参数说明**：
- **host_id**：主机ID，通过引用输入变量host_id进行赋值，必须是已存在且已安装HSS agent的ECS实例ID
- **version**：防护版本，通过引用输入变量protection_version进行赋值，例如"hss.version.enterprise"表示企业版
- **charging_mode**：计费模式，设置为"postPaid"表示按需计费
- **is_wait_host_available**：是否等待主机agent状态变为在线，通过引用输入变量is_wait_host_available进行赋值，默认值为false
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，可选参数，默认值为null

> 注意：HSS主机防护需要依赖已存在的主机，该主机必须已安装HSS agent。如果主机尚未安装HSS agent，需要先为ECS实例安装HSS agent（可通过在创建ECS实例时配置`agent_list = "hss"`来实现）。防护版本参数需要根据实际需求选择，常见版本包括基础版、企业版等。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# HSS主机防护配置
host_id            = "your_host_id"
protection_version = "hss.version.enterprise"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `host_id`需要设置为已存在且已安装HSS agent的ECS实例ID
   - `protection_version`可以设置为"hss.version.enterprise"（企业版）或其他支持的防护版本
   - `is_wait_host_available`可以设置为true，表示等待主机agent状态变为在线后再创建防护
   - `enterprise_project_id`可以设置企业项目ID，如果不需要可以省略
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="host_id=your_host_id" -var="protection_version=hss.version.enterprise"`
2. 环境变量：`export TF_VAR_host_id=your_host_id` 和 `export TF_VAR_protection_version=hss.version.enterprise`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。确保指定的主机ID对应的ECS实例已安装HSS agent，否则防护创建可能会失败。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建HSS主机防护：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建HSS主机防护
4. 运行 `terraform show` 查看已创建的HSS主机防护详情

> 注意：HSS主机防护创建前需要确保指定的主机已安装HSS agent。如果主机尚未安装HSS agent，需要先为ECS实例安装HSS agent。如果设置了`is_wait_host_available = true`，Terraform会等待主机agent状态变为在线后再创建防护，这可能需要一些时间。

## 参考信息

- [华为云HSS产品文档](https://support.huaweicloud.com/hss/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [主机防护最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/hss/postpaid-host-protection)
