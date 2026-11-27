# 部署空闲会话查询与清理

## 应用场景

GaussDB 是华为云提供的高性能、高可用、高安全的企业级分布式关系型数据库服务。在数据库日常运维过程中，长时间处于空闲状态的会话会持续占用连接资源与内存，可能导致连接数耗尽、实例性能下降等问题，因此需要定期识别并清理空闲会话。

本最佳实践将介绍如何使用Terraform查询并清理GaussDB实例的空闲会话。方案先通过数据源查询实例可用的节点与组件信息，再基于查询结果对指定节点执行空闲会话清理操作，帮助运维人员以基础设施即代码的方式自动化完成会话治理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [GaussDB实例节点信息查询（data.huaweicloud_gaussdb_key_view_nodes_deliver）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_key_view_nodes_deliver)

### 资源

- [GaussDB空闲会话查询与清理（huaweicloud_gaussdb_idle_sessions_query_and_kill）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_idle_sessions_query_and_kill)

### 资源/数据源依赖关系

```
data.huaweicloud_gaussdb_key_view_nodes_deliver
    └── huaweicloud_gaussdb_idle_sessions_query_and_kill
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询GaussDB实例的节点信息

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询GaussDB实例的节点信息
variable "instance_id" {
  description = "The ID of the GaussDB instance"
  type        = string
  default     = ""
}

data "huaweicloud_gaussdb_key_view_nodes_deliver" "test" {
  instance_id = var.instance_id
}
```

**参数说明**：
- **instance_id**：GaussDB实例ID，通过引用输入变量 instance_id 进行赋值

### 3. 查询并清理空闲会话

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询并清理GaussDB实例的空闲会话
resource "huaweicloud_gaussdb_idle_sessions_query_and_kill" "test" {
  instance_id  = var.instance_id
  node_id      = data.huaweicloud_gaussdb_key_view_nodes_deliver.test.nodes.0.node_id
  component_id = data.huaweicloud_gaussdb_key_view_nodes_deliver.test.nodes.0.component_id
}
```

**参数说明**：
- **instance_id**：GaussDB实例ID，通过引用输入变量 instance_id 进行赋值
- **node_id**：待清理空闲会话的节点ID，通过引用数据源 `huaweicloud_gaussdb_key_view_nodes_deliver` 查询结果中的节点ID进行赋值
- **component_id**：待清理空闲会话的组件ID，通过引用数据源 `huaweicloud_gaussdb_key_view_nodes_deliver` 查询结果中的组件ID进行赋值

> 说明：仅支持 CN 或 DN（主、备）组件的节点。删除该资源仅将其从Terraform状态中移除，不会撤销已执行的会话清理操作。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# GaussDB实例信息
instance_id = "your_instance_id"
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

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始查询并清理GaussDB实例的空闲会话
4. 运行 `terraform show` 查看已查询并清理的空闲会话信息

## 参考信息

- [华为云GaussDB产品文档](https://support.huaweicloud.com/gaussdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GaussDB空闲会话查询与清理最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/idle-sessions-query-and-kill)
