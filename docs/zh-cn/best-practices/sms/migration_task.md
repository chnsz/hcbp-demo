# 部署迁移任务

## 应用场景

主机迁移服务（Server Migration Service，SMS）是华为云提供的服务器迁移服务，支持将物理服务器、虚拟机或其他云平台的服务器迁移到华为云，实现业务的无缝迁移。迁移任务是SMS服务的核心资源，用于执行具体的服务器迁移操作。通过创建迁移任务，可以配置迁移类型、操作系统类型、源服务器和目标模板等参数，实现服务器的自动化迁移。本最佳实践将介绍如何使用Terraform自动化部署迁移任务，包括查询可用区和源服务器、创建服务器模板和创建迁移任务。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [源服务器列表查询数据源（data.huaweicloud_sms_source_servers）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/sms_source_servers)

### 资源

- [服务器模板资源（huaweicloud_sms_server_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sms_server_template)
- [迁移任务资源（huaweicloud_sms_task）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sms_task)

### 资源/数据源依赖关系

```text
data.huaweicloud_availability_zones
    └── huaweicloud_sms_server_template

data.huaweicloud_sms_source_servers
    └── huaweicloud_sms_task

huaweicloud_sms_server_template
    └── huaweicloud_sms_task
```

> 注意：迁移任务需要依赖源服务器和服务器模板。服务器模板需要指定可用区，用于创建目标服务器。迁移任务通过引用源服务器ID和服务器模板ID来配置迁移参数。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询可用区列表：

```hcl
# 查询可用区列表数据源
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 该数据源无需输入参数，会自动查询当前区域下的所有可用区

> 注意：可用区列表用于后续创建服务器模板时指定可用区。

### 3. 查询源服务器列表

在TF文件（如main.tf）中添加以下脚本以查询源服务器列表：

```hcl
variable "source_server_name" {
  description = "The name of the SMS source server"
  type        = string
  default     = null
}

# 查询源服务器列表数据源
data "huaweicloud_sms_source_servers" "test" {
  name = var.source_server_name
}
```

**参数说明**：
- **name**：源服务器名称，通过引用输入变量 `source_server_name` 进行赋值，可选参数，用于过滤源服务器

> 注意：源服务器列表用于获取需要迁移的源服务器信息，后续创建迁移任务时需要引用源服务器ID。

### 4. 创建服务器模板

在TF文件（如main.tf）中添加以下脚本以创建服务器模板：

```hcl
variable "server_template_name" {
  description = "The name of the SMS server template"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建服务器模板资源
resource "huaweicloud_sms_server_template" "test" {
  name              = var.server_template_name
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
}
```

**参数说明**：
- **name**：服务器模板名称，通过引用输入变量 `server_template_name` 进行赋值
- **availability_zone**：可用区，通过引用可用区列表数据源的第一个可用区名称进行赋值，使用 `try` 函数处理可能的空值

> 注意：服务器模板用于定义目标服务器的配置，包括可用区、规格等信息。创建迁移任务时需要引用服务器模板ID。

### 5. 创建迁移任务

在TF文件（如main.tf）中添加以下脚本以创建迁移任务：

```hcl
variable "migrate_task_type" {
  description = "The type of the SMS task"
  type        = string
}

variable "server_os_type" {
  description = "The OS type of the server"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建迁移任务资源
resource "huaweicloud_sms_task" "test" {
  type             = var.migrate_task_type
  os_type          = var.server_os_type
  source_server_id = try(data.huaweicloud_sms_source_servers.test.servers[0].id, null)
  vm_template_id   = huaweicloud_sms_server_template.test.id
}
```

**参数说明**：
- **type**：迁移任务类型，通过引用输入变量 `migrate_task_type` 进行赋值，支持 `MIGRATE_BLOCK`（块级迁移）和 `MIGRATE_FILE`（文件级迁移）
- **os_type**：服务器操作系统类型，通过引用输入变量 `server_os_type` 进行赋值，支持 `WINDOWS` 和 `LINUX`
- **source_server_id**：源服务器ID，通过引用源服务器列表数据源的第一个服务器ID进行赋值，使用 `try` 函数处理可能的空值
- **vm_template_id**：虚拟机模板ID，通过引用服务器模板资源的ID进行赋值

> 注意：迁移任务用于执行具体的服务器迁移操作。迁移任务类型需要根据实际需求选择，块级迁移适用于整机迁移，文件级迁移适用于文件迁移。源服务器ID和虚拟机模板ID是必填参数。

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 源服务器配置（可选）
source_server_name = "tf_source_server_name"

# 服务器模板配置（必填）
server_template_name = "tf_server_template_name"

# 迁移任务配置（必填）
migrate_task_type = "MIGRATE_BLOCK"
server_os_type    = "WINDOWS"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="server_template_name=tf_server_template_name" -var="migrate_task_type=MIGRATE_BLOCK"`
2. 环境变量：`export TF_VAR_server_template_name=tf_server_template_name`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建服务器模板和迁移任务
4. 运行 `terraform show` 查看已创建的迁移任务

## 参考信息

- [华为云SMS产品文档](https://support.huaweicloud.com/sms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SMS迁移任务最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sms/migration-task)
