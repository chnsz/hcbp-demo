# 部署备份下载授权

## 应用场景

GaussDB是华为云提供的高性能、高可用、高安全的企业级分布式关系型数据库服务。当用户需要将GaussDB实例的备份文件下载到本地进行离线分析、归档或迁移时，需要先对备份进行下载授权，获取备份文件所在的OBS桶名称以及可通过OBS Browser+下载的备份文件路径。

本最佳实践将介绍如何使用Terraform自动化部署GaussDB备份下载授权，包括备份ID的指定以及授权资源的创建，帮助用户快速获取备份文件的下载信息。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [GaussDB备份下载授权（huaweicloud_gaussdb_download_backups_authorize）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_download_backups_authorize)

### 资源/数据源依赖关系

```
huaweicloud_gaussdb_download_backups_authorize
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建GaussDB备份下载授权

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GaussDB备份下载授权资源
variable "backup_id" {
  description = "The backup ID in UUID format (36 characters) that uniquely identifies a GaussDB backup"
  type        = string
}

resource "huaweicloud_gaussdb_download_backups_authorize" "test" {
  backup_id = var.backup_id
}
```

**参数说明**：
- **backup_id**：GaussDB备份ID，通过引用输入变量 backup_id 进行赋值，格式为36位UUID，用于唯一标识需要授权下载的备份

> 说明：该资源为一次性动作资源，创建操作会向GaussDB（opengauss）服务端点发送POST请求完成备份下载授权；资源ID等于backup_id；backup_id参数不支持更新，变更时会触发资源重建；读取和更新操作为空操作；删除该资源仅从Terraform状态中移除，不会在云端撤销授权；创建成功后可通过计算属性`bucket`和`file_paths`获取备份文件所在的OBS桶名称以及可通过OBS Browser+下载的备份文件路径。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
backup_id = "your_gaussdb_backup_id"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="backup_id=your_gaussdb_backup_id"`
2. 环境变量：`export TF_VAR_backup_id=your_gaussdb_backup_id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建GaussDB备份下载授权
4. 运行 `terraform show` 查看已创建的GaussDB备份下载授权

## 参考信息

- [华为云GaussDB产品文档](https://support.huaweicloud.com/gaussdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GaussDB备份下载授权最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-download-backups-authorize)
