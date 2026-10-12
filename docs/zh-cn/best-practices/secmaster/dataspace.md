# 部署数据空间

## 应用场景

安全云脑（SecMaster）是华为云原生的新一代安全运营中心，提供云上资产管理、安全态势管理、安全信息和事件管理、安全编排与自动响应等能力。数据空间（Dataspace）是安全云脑工作空间下的数据存储与隔离单元，用于对工作空间内的安全数据进行分类存放与独立管理。

本最佳实践将介绍如何使用Terraform自动化部署安全云脑数据空间，包括工作空间的创建以及在其下创建数据空间，帮助您快速构建安全云脑的数据存储基础。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [安全云脑工作空间（huaweicloud_secmaster_workspace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_workspace)
- [安全云脑数据空间（huaweicloud_secmaster_dataspace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/secmaster_dataspace)

### 资源/数据源依赖关系

```
huaweicloud_secmaster_workspace
    └── huaweicloud_secmaster_dataspace
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建安全云脑工作空间

在TF文件（如main.tf）中添加以下脚本以创建安全云脑工作空间：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全云脑工作空间资源
variable "workspace_name" {
  description = "The name of the SecMaster workspace"
  type        = string
}

variable "workspace_description" {
  description = "The description of the SecMaster workspace"
  type        = string
  default     = "Created by Terraform"
}

resource "huaweicloud_secmaster_workspace" "test" {
  name         = var.workspace_name
  project_name = var.region_name
  description  = var.workspace_description
}
```

**参数说明**：
- **name**：工作空间名称，通过引用输入变量 workspace_name 进行赋值
- **project_name**：工作空间所属项目名称，通过引用输入变量 region_name 进行赋值
- **description**：工作空间描述，通过引用输入变量 workspace_description 进行赋值

### 3. 创建安全云脑数据空间

在TF文件（如main.tf）中添加以下脚本以创建安全云脑数据空间：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全云脑数据空间资源
variable "dataspace_name" {
  description = "The name of the SecMaster dataspace. The name can only contain English letters, digits and hyphens (-), and cannot start or end with a hyphen (-), nor can they appear consecutively. Valid length: 5-63"
  type        = string
}

variable "dataspace_description" {
  description = "The description of the SecMaster dataspace"
  type        = string
  default     = "Created by Terraform"
}

resource "huaweicloud_secmaster_dataspace" "test" {
  workspace_id   = huaweicloud_secmaster_workspace.test.id
  dataspace_name = var.dataspace_name
  description    = var.dataspace_description
}
```

**参数说明**：
- **workspace_id**：数据空间所属工作空间ID，通过引用工作空间资源 `huaweicloud_secmaster_workspace.test` 的 `id` 进行赋值
- **dataspace_name**：数据空间名称，通过引用输入变量 dataspace_name 进行赋值，名称仅可包含英文字母、数字和连字符（-），不能以连字符开头或结尾，也不能连续出现，有效长度为5-63
- **description**：数据空间描述，通过引用输入变量 dataspace_description 进行赋值

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
workspace_name = "secmaster-workspace-test"
dataspace_name = "secmaster-dataspace-test"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="workspace_name=secmaster-workspace-test"`
2. 环境变量：`export TF_VAR_workspace_name=secmaster-workspace-test`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建安全云脑数据空间
4. 运行 `terraform show` 查看已创建的安全云脑数据空间

## 参考信息

- [华为云安全云脑产品文档](https://support.huaweicloud.com/secmaster/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [安全云脑数据空间最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/secmaster/dataspace)
