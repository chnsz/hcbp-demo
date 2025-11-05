# 部署云应用服务器组

## 应用场景

华为云云桌面（Workspace）是一种基于云计算的桌面虚拟化服务，为企业用户提供安全、便捷的云上办公解决方案。云应用服务器组是Workspace服务的重要组成部分，用于承载各种应用程序，为用户提供统一的应用程序访问体验。

通过云应用服务器组，企业可以实现应用程序的集中部署、统一管理和安全控制，用户可以通过各种终端设备访问云上的应用程序，无需在本地安装和维护软件。本最佳实践将介绍如何使用Terraform自动化部署Workspace云应用服务器组。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [Workspace服务查询数据源（data.huaweicloud_workspace_service）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_service)

### 资源

- [Workspace应用服务器组资源（huaweicloud_workspace_app_server_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_app_server_group)

### 资源/数据源依赖关系

```
data.huaweicloud_workspace_service.test
    └── huaweicloud_workspace_app_server_group.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../docs/introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询Workspace服务信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建云应用服务器组：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的Workspace服务信息，用于创建云应用服务器组
data "huaweicloud_workspace_service" "test" {}
```

**参数说明**：
- 该数据源无需额外参数，会自动查询当前region下的Workspace服务信息

### 3. 创建Workspace云应用服务器组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云应用服务器组资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Workspace云应用服务器组资源
resource "huaweicloud_workspace_app_server_group" "test" {
  name             = var.app_server_group_name
  app_type         = var.app_server_group_app_type
  os_type          = var.app_server_group_os_type
  flavor_id        = var.app_server_group_flavor_id
  image_type       = "gold"
  image_id         = var.app_server_group_image_id
  image_product_id = var.app_server_group_image_product_id
  vpc_id           = data.huaweicloud_workspace_service.test.vpc_id
  subnet_id        = try(data.huaweicloud_workspace_service.test.network_ids[0], null)
  system_disk_type = var.app_server_group_system_disk_type
  system_disk_size = var.app_server_group_system_disk_size
  is_vdi           = true
}
```

**参数说明**：
- **name**：云应用服务器组的名称，通过引用输入变量app_server_group_name进行赋值
- **app_type**：应用类型，通过引用输入变量app_server_group_app_type进行赋值，默认为"SESSION_DESKTOP_APP"
- **os_type**：操作系统类型，通过引用输入变量app_server_group_os_type进行赋值，默认为"Windows"
- **flavor_id**：规格ID，通过引用输入变量app_server_group_flavor_id进行赋值
- **image_type**：镜像类型，固定设置为"gold"（黄金镜像）
- **image_id**：镜像ID，通过引用输入变量app_server_group_image_id进行赋值
- **image_product_id**：镜像产品ID，通过引用输入变量app_server_group_image_product_id进行赋值
- **vpc_id**：VPC ID，根据Workspace服务查询数据源（data.huaweicloud_workspace_service）的返回结果进行赋值
- **subnet_id**：子网ID，根据Workspace服务查询数据源（data.huaweicloud_workspace_service）的返回结果进行赋值
- **system_disk_type**：系统盘类型，通过引用输入变量app_server_group_system_disk_type进行赋值，默认为"SAS"
- **system_disk_size**：系统盘大小，通过引用输入变量app_server_group_system_disk_size进行赋值，默认为80GB
- **is_vdi**：是否为VDI模式，固定设置为true

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 云应用服务器组基本信息
app_server_group_name             = "tf_test_server_group"
app_server_group_app_type         = "SESSION_DESKTOP_APP"
app_server_group_os_type          = "Windows"
app_server_group_flavor_id        = "workspace.appstream.general.xlarge.4"
app_server_group_image_id         = "2ac7b1fb-b198-422b-a45f-61ea285cb6e7"
app_server_group_image_product_id = "OFFI886188719633408000"
app_server_group_system_disk_type = "SAS"
app_server_group_system_disk_size = 80
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="app_server_group_name=my-server-group" -var="app_server_group_flavor_id=workspace.appstream.general.xlarge.4"`
2. 环境变量：`export TF_VAR_app_server_group_name=my-server-group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建云应用服务器组
4. 运行 `terraform show` 查看已创建的云应用服务器组

## 参考信息

- [华为云Workspace产品文档](https://support.huaweicloud.com/workspace/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Workspace云应用服务器组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/app/server_group)
