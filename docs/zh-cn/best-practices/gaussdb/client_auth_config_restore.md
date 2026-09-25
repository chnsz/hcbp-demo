# 部署客户端接入认证配置恢复

## 应用场景

GaussDB是华为云提供的高性能、高可用、高安全的企业级分布式关系型数据库服务，支持集中式和分布式部署形态，广泛应用于金融、政企、电商等对数据一致性和可靠性要求较高的核心业务场景。GaussDB通过客户端接入认证（HBA）配置控制允许访问数据库的客户端地址、认证方式与数据库用户，是数据库安全防护的重要一环。

本最佳实践将介绍如何使用Terraform自动化恢复GaussDB实例的客户端接入认证配置，包括指定实例、选择历史记录版本或恢复默认配置等操作。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [GaussDB客户端接入认证配置恢复（huaweicloud_gaussdb_client_auth_config_restore）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_client_auth_config_restore)

### 资源/数据源依赖关系

```
huaweicloud_gaussdb_client_auth_config_restore
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 恢复客户端接入认证配置

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下恢复GaussDB实例的客户端接入认证配置
variable "instance_id" {
  description = "The ID of the GaussDB instance"
  type        = string
  default     = ""
}

variable "hba_history_id" {
  description = "The client access authentication modification history record ID"
  type        = string
  default     = ""
}

resource "huaweicloud_gaussdb_client_auth_config_restore" "test" {
  instance_id    = var.instance_id
  hba_history_id = var.hba_history_id
}
```

**参数说明**：

- **instance_id**：GaussDB实例ID，通过引用输入变量 instance_id 进行赋值
- **hba_history_id**：客户端接入认证修改历史记录ID，通过引用输入变量 hba_history_id 进行赋值；若为空，则表示恢复到默认配置

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证参数
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 资源参数
instance_id    = "your_instance_id"
hba_history_id = "your_hba_history_id"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="instance_id=my-instance-id"`
2. 环境变量：`export TF_VAR_instance_id=my-instance-id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来恢复客户端接入认证配置：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始恢复客户端接入认证配置
4. 运行 `terraform show` 查看已恢复的客户端接入认证配置

## 参考信息

- [华为云GaussDB产品文档](https://support.huaweicloud.com/gaussdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GaussDB客户端接入认证配置恢复最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/client-auth-config-restore)
