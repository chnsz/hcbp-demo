# 部署私有模块及模块版本

## 应用场景

资源编排服务（Resource Formation Service，RFS）是华为云提供的基础设施即代码（Infrastructure as Code，IaC）服务，支持通过模板对云上资源进行编排与自动化部署。私有模块可将一组可复用的模板封装为标准化模块，配合模块版本实现模板的版本化管理，便于在团队与项目之间共享和复用基础设施代码。

本最佳实践将介绍如何使用Terraform创建RFS私有模块，并基于存储在OBS中的模块包创建对应的模块版本，实现私有模块及其版本的自动化部署与管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [RFS私有模块（huaweicloud_rfs_private_module）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_private_module)
- [RFS私有模块版本（huaweicloud_rfs_private_module_version）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_private_module_version)

### 资源/数据源依赖关系

```
huaweicloud_rfs_private_module
    └── huaweicloud_rfs_private_module_version
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建RFS私有模块

在TF文件（如main.tf）中添加以下脚本以创建RFS私有模块：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RFS私有模块资源
variable "module_name" {
  description = "The name of the RFS private module"
  type        = string
}

variable "module_description" {
  description = "The description of the RFS private module"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_private_module" "test" {
  module_name        = var.module_name
  module_description = var.module_description
}
```

**参数说明**：
- **module_name**：私有模块的名称，通过引用输入变量 module_name 进行赋值，需在当前账号与区域内唯一，仅支持字母、数字、下划线（_）和中划线（-），且必须以字母开头
- **module_description**：私有模块的描述，通过引用输入变量 module_description 进行赋值，该参数为可选参数，一旦设置后不可更新为空值

### 3. 创建RFS私有模块版本

在TF文件（如main.tf）中添加以下脚本以创建RFS私有模块版本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RFS私有模块版本资源
variable "module_version" {
  description = "The version number of the RFS private module"
  type        = string
}

variable "module_uri" {
  description = "The OBS address of the private module package"
  type        = string
}

variable "version_description" {
  description = "The description of the private module version"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_private_module_version" "test" {
  module_name         = huaweicloud_rfs_private_module.test.module_name
  module_version      = var.module_version
  module_uri          = var.module_uri
  version_description = var.version_description
}
```

**参数说明**：
- **module_name**：私有模块的名称，通过引用前一步创建的私有模块的 module_name 属性进行赋值
- **module_version**：私有模块的版本号，通过引用输入变量 module_version 进行赋值，该参数不可更新，修改后将重建模块版本
- **module_uri**：私有模块包的OBS地址，通过引用输入变量 module_uri 进行赋值，该参数不可更新，修改后将重建模块版本
- **version_description**：私有模块版本的描述，通过引用输入变量 version_description 进行赋值，该参数为可选参数且不可更新

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 私有模块配置
module_name        = "tf-test-rfs-module"
module_description = "An example RFS private module"

# 私有模块版本配置
module_version      = "1.0.0"
module_uri          = "https://your-bucket.obs.cn-north-4.myhuaweicloud.com/tf-test-rfs-module.zip"
version_description = "The first version of the RFS private module"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="module_name=my-module"`
2. 环境变量：`export TF_VAR_module_name=my-module`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建RFS私有模块及模块版本
4. 运行 `terraform show` 查看已创建的RFS私有模块及模块版本

## 参考信息

- [华为云资源编排服务（RFS）产品文档](https://support.huaweicloud.com/rfs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RFS私有模块及模块版本最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rfs/private-module-with-version)
