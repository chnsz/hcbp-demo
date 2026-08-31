# 部署数据库连接

## 应用场景

数据管理服务（Data Admin Service，简称DAS）是用来登录和操作华为云上数据库的Web服务，提供数据库开发、运维、智能诊断的一站式云上数据库管理平台。通过创建数据库实例连接，您可以在DAS中安全地接入已有RDS等数据库实例，并进行可视化SQL操作与运维管理。结合数据库用户创建和连接共享能力，可实现多用户协作访问同一连接资源。本最佳实践将介绍如何使用Terraform自动化部署DAS数据库实例连接，包括创建数据库实例连接、数据库用户，以及将连接共享给其他IAM用户。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [数据库实例连接资源（huaweicloud_das_database_instance_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_database_instance_connection)
- [数据库用户资源（huaweicloud_das_database_user）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_database_user)
- [共享连接资源（huaweicloud_das_shared_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_shared_connection)

### 资源/数据源依赖关系

```text
huaweicloud_das_database_instance_connection
    └── huaweicloud_das_shared_connection

huaweicloud_das_database_user
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建数据库实例连接

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建数据库实例连接资源：

```hcl
variable "connection_instance_id" {
  description = "The ID of the RDS instance to connect"
  type        = string
}

variable "connection_engine_type" {
  description = "The engine type of the database instance"
  type        = string
}

variable "connection_network_type" {
  description = "The network type of the database instance connection"
  type        = string
}

variable "connection_username" {
  description = "The username for the database instance connection"
  type        = string
}

variable "connection_password" {
  description = "The password for the database instance connection"
  type        = string
  sensitive   = true
}

variable "connection_is_save_password" {
  description = "Whether to save the password for the database instance connection"
  type        = bool
  default     = true
  sensitive   = true
}

variable "connection_port" {
  description = "The port of the database instance connection"
  type        = number
  default     = null
}

variable "connection_database_name" {
  description = "The database name of the database instance connection"
  type        = string
  default     = null
}

variable "connection_sql_record_flag" {
  description = "Whether SQL recording is enabled for the database instance connection"
  type        = bool
  default     = null
}

variable "connection_description" {
  description = "The description of the database instance connection"
  type        = string
  default     = null
}

variable "connection_node_ids" {
  description = "The unique identifiers of the instance nodes"
  type        = list(string)
  default     = []
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建数据库实例连接资源
resource "huaweicloud_das_database_instance_connection" "test" {
  instance_id      = var.connection_instance_id
  engine_type      = var.connection_engine_type
  network_type     = var.connection_network_type
  username         = var.connection_username
  password         = var.connection_password
  is_save_password = var.connection_is_save_password

  port            = var.connection_port
  database_name   = var.connection_database_name
  sql_record_flag = var.connection_sql_record_flag
  description     = var.connection_description
  node_ids        = var.connection_node_ids
}
```

**参数说明**：
- **instance_id**：待连接的RDS实例ID，通过引用输入变量 `connection_instance_id` 进行赋值
- **engine_type**：数据库实例的引擎类型，通过引用输入变量 `connection_engine_type` 进行赋值
- **network_type**：数据库实例连接的网络类型，通过引用输入变量 `connection_network_type` 进行赋值
- **username**：数据库实例连接的用户名，通过引用输入变量 `connection_username` 进行赋值
- **password**：数据库实例连接的密码，通过引用输入变量 `connection_password` 进行赋值
- **is_save_password**：是否保存数据库实例连接的密码，通过引用输入变量 `connection_is_save_password` 进行赋值
- **port**：数据库实例连接的端口，通过引用输入变量 `connection_port` 进行赋值
- **database_name**：数据库实例连接的数据库名称，通过引用输入变量 `connection_database_name` 进行赋值
- **sql_record_flag**：是否为数据库实例连接启用SQL记录，通过引用输入变量 `connection_sql_record_flag` 进行赋值
- **description**：数据库实例连接的描述信息，通过引用输入变量 `connection_description` 进行赋值
- **node_ids**：实例节点的唯一标识列表，通过引用输入变量 `connection_node_ids` 进行赋值

> 注意：部署前需已存在可连接的RDS等数据库实例。请妥善保管数据库密码等敏感信息，避免将其提交到版本控制系统。

### 3. 创建数据库用户

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建数据库用户资源：

```hcl
variable "db_user_name" {
  description = "The name of the database user"
  type        = string
}

variable "db_user_password" {
  description = "The password of the database user"
  type        = string
  sensitive   = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建数据库用户资源
resource "huaweicloud_das_database_user" "test" {
  instance_id = var.connection_instance_id
  name        = var.db_user_name
  password    = var.db_user_password
}
```

**参数说明**：
- **instance_id**：数据库用户所属的RDS实例ID，通过引用输入变量 `connection_instance_id` 进行赋值
- **name**：数据库用户名称，通过引用输入变量 `db_user_name` 进行赋值
- **password**：数据库用户密码，通过引用输入变量 `db_user_password` 进行赋值

### 4. 创建共享连接

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建共享连接资源：

```hcl
variable "shared_user_id" {
  description = "The IAM user ID to share the connection with"
  type        = string
}

variable "shared_user_name" {
  description = "The IAM user name to share the connection with"
  type        = string
}

variable "shared_expired_at" {
  description = "The expiration time of the shared connection, in RFC3339 format"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建共享连接资源
resource "huaweicloud_das_shared_connection" "test" {
  connection_id = huaweicloud_das_database_instance_connection.test.id
  user_id       = var.shared_user_id
  user_name     = var.shared_user_name
  expired_at    = var.shared_expired_at
}
```

**参数说明**：
- **connection_id**：待共享的数据库实例连接ID，通过引用数据库实例连接资源的ID进行赋值
- **user_id**：共享目标IAM用户ID，通过引用输入变量 `shared_user_id` 进行赋值
- **user_name**：共享目标IAM用户名，通过引用输入变量 `shared_user_name` 进行赋值
- **expired_at**：共享连接的过期时间（RFC3339格式），通过引用输入变量 `shared_expired_at` 进行赋值

> 注意：共享连接依赖已创建的数据库实例连接。请确认目标IAM用户信息正确，并按需配置共享过期时间。

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 数据库实例连接配置
connection_instance_id     = "tf_test_das_instance_id"
connection_engine_type     = "mysql"
connection_network_type    = "rds"
connection_username        = "tf_test_username"
connection_password        = "tf_test_password"
connection_is_save_password = true
connection_port            = 3306
connection_database_name   = "tf_test_database"
connection_sql_record_flag = true
connection_description     = "tf_test_connection_description"
connection_node_ids        = []

# 数据库用户配置
db_user_name     = "tf_test_db_user"
db_user_password = "tf_test_db_password"

# 共享连接配置
shared_user_id   = "tf_test_shared_user_id"
shared_user_name = "tf_test_shared_user_name"
shared_expired_at = null
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="connection_instance_id=your_rds_instance_id" -var="connection_username=your_username"`
2. 环境变量：`export TF_VAR_connection_instance_id=your_rds_instance_id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建数据库连接
4. 运行 `terraform show` 查看已创建的数据库连接

## 参考信息

- [华为云DAS产品文档](https://support.huaweicloud.com/das/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DAS数据库连接最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/das/database-connection)
