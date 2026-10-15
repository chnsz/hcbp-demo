# 部署客户端接入认证配置

## 应用场景

GaussDB是华为云提供的高性能、高可用、高安全的企业级分布式关系型数据库服务，支持通过客户端接入认证（HBA，Host-Based Authentication）配置对数据库的访问来源进行精细化管控。客户端接入认证配置定义了允许连接数据库的客户端地址范围、匹配的数据库与用户，以及所采用的认证方式，是保障数据库访问安全的重要手段。

本最佳实践将介绍如何使用Terraform自动化部署GaussDB客户端接入认证配置，包括指定GaussDB实例、配置客户端连接类型、匹配的数据库与用户、IP地址范围以及认证方式。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [GaussDB客户端接入认证配置（huaweicloud_gaussdb_client_auth_config）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_client_auth_config)

### 资源/数据源依赖关系

```
huaweicloud_gaussdb_client_auth_config
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建GaussDB客户端接入认证配置

在TF文件（如main.tf）中添加以下脚本以创建GaussDB客户端接入认证配置：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GaussDB客户端接入认证配置资源
variable "instance_id" {
  description = "The ID of the GaussDB instance"
  type        = string
  default     = ""
}

variable "config_type" {
  description = "The client connection type. Valid values are host, hostssl, and hostnossl"
  type        = string
  default     = "host"
}

variable "config_database" {
  description = "The database name that the record matches. The value can be all or an existing database name"
  type        = string
  default     = "all"
}

variable "config_user" {
  description = "The database user that the record matches. The value can be all or an existing username"
  type        = string
  default     = "root"
}

variable "config_address" {
  description = "The IP address range that the record matches. The value must be in CIDR format"
  type        = string
  default     = "10.10.0.0/16"
}

variable "config_method" {
  description = "The authentication method. Valid values include md5, sha256, sm3, reject, cert, etc"
  type        = string
  default     = "md5"
}

resource "huaweicloud_gaussdb_client_auth_config" "test" {
  instance_id = var.instance_id
  type        = var.config_type
  database    = var.config_database
  user        = var.config_user
  address     = var.config_address
  method      = var.config_method
}
```

**参数说明**：

- **instance_id**：GaussDB实例ID，通过引用输入变量 instance_id 进行赋值
- **type**：客户端连接类型，通过引用输入变量 config_type 进行赋值，可选值为 `host`（允许SSL和非SSL连接）、`hostssl`（仅允许SSL连接）、`hostnossl`（仅允许非SSL连接）
- **database**：该记录匹配的数据库名称，通过引用输入变量 config_database 进行赋值，取值可以是 `all` 或已存在的数据库名称
- **user**：该记录匹配的数据库用户，通过引用输入变量 config_user 进行赋值，取值可以是 `all` 或已存在的用户名
- **address**：该记录匹配的IP地址范围，通过引用输入变量 config_address 进行赋值，必须为CIDR格式（如 `10.10.0.0/16`）
- **method**：认证方式，通过引用输入变量 config_method 进行赋值，可选值包括 `md5`、`sha256`、`sm3`、`reject`、`cert` 等

> 注意：`instance_id`、`type`、`database`、`user` 和 `address` 在创建后不可修改，变更这些参数将导致资源重建；仅 `method` 字段支持创建后原地更新。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
instance_id     = "your_instance_id"
config_type     = "host"
config_database = "all"
config_user     = "root"
config_address  = "10.10.0.0/16"
config_method   = "md5"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="instance_id=your_instance_id"`
2. 环境变量：`export TF_VAR_instance_id=your_instance_id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建GaussDB客户端接入认证配置
4. 运行 `terraform show` 查看已创建的GaussDB客户端接入认证配置

## 参考信息

- [华为云GaussDB产品文档](https://support.huaweicloud.com/gaussdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GaussDB客户端接入认证配置最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-client-auth-config)
