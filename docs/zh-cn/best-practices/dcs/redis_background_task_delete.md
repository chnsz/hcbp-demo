# 部署Redis后台任务删除

## 应用场景

分布式缓存服务（Distributed Cache Service，DCS）是华为云提供的高性能、高可用的内存数据库服务，支持Redis、Memcached等主流缓存引擎。在使用DCS Redis实例的过程中，系统会在实例创建、扩容、配置变更、备份、恢复等操作时生成后台任务，用于跟踪和管理这些耗时操作的执行状态。

本最佳实践将介绍如何使用Terraform删除DCS Redis实例中指定的后台任务。通过调用华为云DCS服务的删除后台任务接口，您可以清理不再需要的后台任务记录，保持实例任务列表的整洁。本实践针对已存在的DCS实例和后台任务进行操作，不创建任何新的基础设施。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DCS后台任务删除（huaweicloud_dcs_background_task_delete）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_background_task_delete)

### 资源/数据源依赖关系

```
huaweicloud_dcs_background_task_delete
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 删除DCS后台任务

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下删除DCS实例的后台任务
variable "dcs_instance_id" {
  description = "The ID of the DCS instance that owns the background task"
  type        = string
  default     = ""
}

variable "background_task_id" {
  description = "The ID of the background task to delete"
  type        = string
  default     = ""
}

resource "huaweicloud_dcs_background_task_delete" "test" {
  instance_id = var.dcs_instance_id
  task_id     = var.background_task_id
}
```

**参数说明**：
- **instance_id**：通过引用输入变量 dcs_instance_id 进行赋值，指定拥有该后台任务的DCS实例ID。
- **task_id**：通过引用输入变量 background_task_id 进行赋值，指定要删除的后台任务ID。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
dcs_instance_id    = "your_dcs_instance_id"
background_task_id = "your_background_task_id"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="dcs_instance_id=my-instance-id"`
2. 环境变量：`export TF_VAR_dcs_instance_id=my-instance-id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来删除后台任务：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源变更计划
3. 确认资源计划无误后，运行 `terraform apply` 开始删除指定的后台任务
4. 运行 `terraform show` 查看已删除的后台任务信息

## 参考信息

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [分布式缓存服务Redis后台任务删除最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-background-task-delete)
