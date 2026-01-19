# 部署迁移项目

## 应用场景

主机迁移服务（Server Migration Service，SMS）是华为云提供的服务器迁移服务，支持将物理服务器、虚拟机或其他云平台的服务器迁移到华为云，实现业务的无缝迁移。迁移项目是SMS服务的基础资源，用于管理和组织迁移任务。通过创建迁移项目，可以配置迁移区域、网络类型、服务器类型和同步策略等参数，为后续的迁移任务提供基础配置。本最佳实践将介绍如何使用Terraform自动化部署迁移项目，包括项目基本信息、区域配置、网络配置和同步策略配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [迁移项目资源（huaweicloud_sms_migration_project）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sms_migration_project)

### 资源/数据源依赖关系

```text
huaweicloud_sms_migration_project
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建迁移项目

在TF文件（如main.tf）中添加以下脚本以创建迁移项目：

```hcl
variable "migration_project_name" {
  description = "The migration project name"
  type        = string
}

variable "migration_project_region" {
  description = "The region name"
  type        = string
}

variable "migration_project_use_public_ip" {
  description = "Whether to use a public IP address for migration"
  type        = bool
}

variable "migration_project_exist_server" {
  description = "Whether the server already exists"
  type        = bool
}

variable "migration_project_type" {
  description = "The migration project typ"
  type        = string
}

variable "migration_project_syncing" {
  description = "Whether to continue syncing after the first copy or sync"
  type        = bool
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建迁移项目资源
resource "huaweicloud_sms_migration_project" "test" {
  name          = var.migration_project_name
  region        = var.migration_project_region
  use_public_ip = var.migration_project_use_public_ip
  exist_server  = var.migration_project_exist_server
  type          = var.migration_project_type
  syncing       = var.migration_project_syncing
}
```

**参数说明**：
- **name**：迁移项目名称，通过引用输入变量 `migration_project_name` 进行赋值
- **region**：区域名称，通过引用输入变量 `migration_project_region` 进行赋值，用于指定迁移目标区域
- **use_public_ip**：是否使用公网IP进行迁移，通过引用输入变量 `migration_project_use_public_ip` 进行赋值
- **exist_server**：服务器是否已存在，通过引用输入变量 `migration_project_exist_server` 进行赋值
- **type**：迁移项目类型，通过引用输入变量 `migration_project_type` 进行赋值
- **syncing**：是否在首次复制或同步后继续同步，通过引用输入变量 `migration_project_syncing` 进行赋值

> 注意：迁移项目用于管理和组织迁移任务，需要根据实际的迁移场景配置相应的参数。如果使用公网IP进行迁移，需要确保源服务器可以访问公网。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 迁移项目基本信息（必填）
migration_project_name          = "tf_test_sms_migration_project"
migration_project_region        = "cn-north-4"
migration_project_use_public_ip = true
migration_project_exist_server  = true
migration_project_type          = "LINUX"
migration_project_syncing       = true
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="migration_project_name=tf_test_sms_migration_project" -var="migration_project_region=cn-north-4"`
2. 环境变量：`export TF_VAR_migration_project_name=tf_test_sms_migration_project`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建迁移项目
4. 运行 `terraform show` 查看已创建的迁移项目

## 参考信息

- [华为云SMS产品文档](https://support.huaweicloud.com/sms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SMS迁移项目最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sms/migration-project)
