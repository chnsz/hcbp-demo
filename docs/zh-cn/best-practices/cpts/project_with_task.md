# 部署CPTS项目与测试任务

## 应用场景

云性能测试服务（Cloud Performance Test Service，CPTS）是华为云提供的性能测试服务，帮助用户模拟真实业务场景，对云上应用进行压力测试，以评估系统的性能瓶颈和稳定性。

本最佳实践将介绍如何使用Terraform创建CPTS项目与测试任务。CPTS项目用于组织和管理测试任务，测试任务则定义了压测的并发数等参数。通过Terraform的自动化部署，您可以快速创建CPTS项目与测试任务，为后续的性能测试做好准备。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [CPTS项目（huaweicloud_cpts_project）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cpts_project)
- [CPTS测试任务（huaweicloud_cpts_task）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cpts_task)

### 资源/数据源依赖关系

```
huaweicloud_cpts_project
    └── huaweicloud_cpts_task
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建CPTS项目

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CPTS项目
variable "cpts_project_name" {
  description = "The name of the CPTS project"
  type        = string
}

variable "cpts_project_description" {
  description = "The description of the CPTS project"
  type        = string
  default     = ""
}

resource "huaweicloud_cpts_project" "test" {
  name        = var.cpts_project_name
  description = var.cpts_project_description
}
```

**参数说明**：
- **name**：通过引用输入变量 cpts_project_name 进行赋值，用于指定CPTS项目的名称。
- **description**：通过引用输入变量 cpts_project_description 进行赋值，用于描述CPTS项目。

### 3. 创建CPTS测试任务

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CPTS测试任务
variable "cpts_task_name" {
  description = "The name of the CPTS test task"
  type        = string
}

variable "cpts_task_benchmark_concurrency" {
  description = "The benchmark concurrency of the CPTS test task"
  type        = number
  default     = 200
}

resource "huaweicloud_cpts_task" "test" {
  name                  = var.cpts_task_name
  project_id            = huaweicloud_cpts_project.test.id
  benchmark_concurrency = var.cpts_task_benchmark_concurrency
}
```

**参数说明**：
- **name**：通过引用输入变量 cpts_task_name 进行赋值，用于指定CPTS测试任务的名称。
- **project_id**：通过引用 huaweicloud_cpts_project.test.id 进行赋值，用于指定该测试任务所属的CPTS项目ID。
- **benchmark_concurrency**：通过引用输入变量 cpts_task_benchmark_concurrency 进行赋值，用于指定测试任务的基准并发数。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
region_name                     = "cn-north-4"
access_key                      = "your-access-key"
secret_key                      = "your-secret-key"
cpts_project_name               = "tf_test_cpts_project"
cpts_project_description        = "Created by Terraform for CPTS best practice"
cpts_task_name                  = "tf_test_cpts_task"
cpts_task_benchmark_concurrency = 200
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="cpts_project_name=my-project"`
2. 环境变量：`export TF_VAR_cpts_project_name=my-project`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CPTS项目与测试任务
4. 运行 `terraform show` 查看已创建的CPTS项目与测试任务

## 参考信息

- [华为云云性能测试服务产品文档](https://support.huaweicloud.com/cpts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CPTS项目与测试任务最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cpts/project-with-task)
