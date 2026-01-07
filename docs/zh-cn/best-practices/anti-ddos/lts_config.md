# 部署LTS配置

## 应用场景

Anti-DDoS（Anti-Distributed Denial of Service）是华为云提供的分布式拒绝服务攻击防护服务，能够有效防护针对公网IP的DDoS攻击，保障业务的稳定运行。通过配置Anti-DDoS与云日志服务（LTS）的集成，可以将Anti-DDoS的攻击日志实时传输到LTS，便于进行日志分析、审计和监控。LTS配置可以帮助用户集中管理和分析Anti-DDoS防护日志，及时发现和应对安全威胁。

本最佳实践将介绍如何使用Terraform自动化部署LTS配置，包括创建LTS日志组、日志流，以及配置Anti-DDoS与LTS的集成。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [云日志服务日志组资源（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [云日志服务日志流资源（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [Anti-DDoS LTS配置资源（huaweicloud_antiddos_lts_config）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/antiddos_lts_config)

### 资源/数据源依赖关系

```
huaweicloud_lts_group
    └── huaweicloud_lts_stream
        └── huaweicloud_antiddos_lts_config
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建云日志服务日志组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云日志服务日志组资源：

```hcl
variable "lts_group_name" {
  description = "The name of the LTS group"
  type        = string
}

variable "lts_ttl_in_days" {
  description = "The log expiration time(days)"
  type        = number
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云日志服务日志组资源
resource "huaweicloud_lts_group" "test" {
  group_name            = var.lts_group_name
  ttl_in_days           = var.lts_ttl_in_days
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **group_name**：日志组名称，通过引用输入变量lts_group_name进行赋值
- **ttl_in_days**：日志保存时间（单位：天），通过引用输入变量lts_ttl_in_days进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null

### 3. 创建云日志服务日志流资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云日志服务日志流资源：

```hcl
variable "lts_stream_name" {
  description = "The name of the LTS stream"
  type        = string
}

variable "lts_is_favorite" {
  description = "Whether to favorite the log stream"
  type        = bool
  default     = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云日志服务日志流资源
resource "huaweicloud_lts_stream" "test" {
  group_id              = huaweicloud_lts_group.test.id
  stream_name           = var.lts_stream_name
  is_favorite           = var.lts_is_favorite
  enterprise_project_id  = var.enterprise_project_id
}
```

**参数说明**：
- **group_id**：日志组ID，引用前面创建的云日志服务日志组资源（huaweicloud_lts_group.test）的ID
- **stream_name**：日志流名称，通过引用输入变量lts_stream_name进行赋值
- **is_favorite**：是否收藏日志流，通过引用输入变量lts_is_favorite进行赋值，默认值为false
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null

### 4. 创建Anti-DDoS LTS配置资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建Anti-DDoS LTS配置资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Anti-DDoS LTS配置资源
resource "huaweicloud_antiddos_lts_config" "test" {
  lts_group_id          = huaweicloud_lts_group.test.id
  lts_attack_stream_id   = huaweicloud_lts_stream.test.id
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **lts_group_id**：LTS日志组ID，引用前面创建的云日志服务日志组资源（huaweicloud_lts_group.test）的ID
- **lts_attack_stream_id**：LTS攻击日志流ID，引用前面创建的云日志服务日志流资源（huaweicloud_lts_stream.test）的ID
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为null

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 云日志服务日志组配置
lts_group_name  = "test-lts-group-name"
lts_ttl_in_days  = 7

# 云日志服务日志流配置
lts_stream_name  = "test-lts-stream-name"
lts_is_favorite  = true

# 企业项目配置
enterprise_project_id = "0"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="lts_group_name=test-group" -var="lts_stream_name=test-stream"`
2. 环境变量：`export TF_VAR_lts_group_name=test-group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Anti-DDoS LTS配置
4. 运行 `terraform show` 查看已创建的Anti-DDoS LTS配置详情

## 参考信息

- [华为云Anti-DDoS产品文档](https://support.huaweicloud.com/antiddos/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Anti-DDoS LTS配置最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/antiddos/lts-config)
