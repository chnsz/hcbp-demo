# 部署MongoDB分片连接

## 应用场景

数据复制服务（Data Replication Service，DRS）是华为云提供的一站式数据复制服务，支持数据库的实时迁移、实时同步和实时灾备，帮助用户在云上或云下数据库之间实现数据的高效流转。DRS通过连接管理功能，将源数据库与目标数据库的接入信息统一维护，为后续的迁移、同步和灾备任务提供基础。

本最佳实践将介绍如何使用Terraform自动化创建DRS连接，用于接入自建的MongoDB分片集群数据库。该连接包含主节点接入信息、多个分片节点接入信息、SSL配置以及驱动配置，为后续基于该连接开展的数据复制任务奠定基础。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DRS连接（huaweicloud_drs_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_connection)

### 资源/数据源依赖关系

```
huaweicloud_drs_connection
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DRS连接

在TF文件（如main.tf）中添加以下脚本以创建DRS连接：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DRS连接资源
variable "connection_name" {
  description = "The DRS connection name"
  type        = string
}

variable "description" {
  description = "The description of the DRS connection"
  type        = string
  default     = ""
}

variable "endpoint_ip" {
  description = "The IP address and port of the primary MongoDB database, e.g. 192.168.0.1:8080"
  type        = string
}

variable "db_user" {
  description = "The database username"
  type        = string
  default     = "mog"
}

variable "db_password" {
  description = "The password for the MongoDB database user"
  type        = string
  sensitive   = true
}

variable "db_name" {
  description = "The database name"
  type        = string
  default     = "root"
}

variable "shard1_ip" {
  description = "The IP address and port of the first MongoDB shard, e.g. 192.168.0.1:8000"
  type        = string
}

variable "shard2_ip" {
  description = "The IP address and port of the second MongoDB shard, e.g. 192.168.0.2:8000"
  type        = string
}

variable "driver_name" {
  description = "The driver name of the connection configuration"
  type        = string
  default     = "mongodb"
}

resource "huaweicloud_drs_connection" "test" {
  name        = var.connection_name
  db_type     = "mongodb"
  description = var.description

  endpoint {
    endpoint_name = "mongodb"
    ip            = var.endpoint_ip
    db_user       = var.db_user
    db_password   = var.db_password
    db_name       = var.db_name

    source_sharding {
      endpoint_name = "mongodb"
      ip            = var.shard1_ip
      db_user       = var.db_user
      db_password   = var.db_password
      db_name       = var.db_name
    }

    source_sharding {
      endpoint_name = "mongodb"
      ip            = var.shard2_ip
      db_user       = var.db_user
      db_password   = var.db_password
      db_name       = var.db_name
    }
  }

  ssl {
    ssl_link = false
  }

  config {
    driver_name = var.driver_name
  }

  lifecycle {
    ignore_changes = [
      endpoint.0.db_password,
      endpoint.0.source_sharding.0.db_password,
      endpoint.0.source_sharding.0.endpoint_name,
      endpoint.0.source_sharding.1.db_password,
      endpoint.0.source_sharding.1.endpoint_name,
    ]
  }
}
```

**参数说明**：
- **name**：DRS连接名称，通过引用输入变量 connection_name 进行赋值
- **db_type**：数据库类型，固定为 mongodb
- **description**：DRS连接描述，通过引用输入变量 description 进行赋值
- **endpoint.endpoint_name**：源数据库的端点名称，固定为 mongodb
- **endpoint.ip**：主节点MongoDB数据库的IP地址和端口，通过引用输入变量 endpoint_ip 进行赋值，格式为 `ip:port`
- **endpoint.db_user**：数据库用户名，通过引用输入变量 db_user 进行赋值
- **endpoint.db_password**：数据库用户密码，通过引用输入变量 db_password 进行赋值
- **endpoint.db_name**：数据库名称，通过引用输入变量 db_name 进行赋值
- **endpoint.source_sharding**：分片节点接入信息，本实践配置两个分片节点，分别通过引用输入变量 shard1_ip 和 shard2_ip 进行赋值
- **ssl.ssl_link**：是否启用SSL连接，本实践设置为 false
- **config.driver_name**：连接配置的驱动名称，通过引用输入变量 driver_name 进行赋值
- **lifecycle.ignore_changes**：由于部分属性（如数据库密码、分片端点名称）不会被API返回，需忽略这些属性的变更以避免不必要的重建

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
connection_name = "your_drs_mongodb_connection"
db_password     = "Test@123456"
endpoint_ip     = "192.168.0.1:8080"
shard1_ip       = "192.168.0.1:8000"
shard2_ip       = "192.168.0.2:8000"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="connection_name=my-connection"`
2. 环境变量：`export TF_VAR_connection_name=my-connection`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DRS连接
4. 运行 `terraform show` 查看已创建的DRS连接

## 参考信息

- [华为云数据复制服务产品文档](https://support.huaweicloud.com/drs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DRS MongoDB分片连接最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/drs-connection-mongodb)
