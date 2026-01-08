# 部署中心网络

## 应用场景

云连接（Cloud Connect，CC）中心网络是云连接服务提供的核心网络管理功能，用于统一管理和连接多个网络实例。通过创建中心网络，您可以构建一个集中式的网络架构，实现跨区域、跨网络的统一管理和互联。通过Terraform自动化创建中心网络，可以确保网络资源管理的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建云连接中心网络。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [云连接中心网络资源（huaweicloud_cc_central_network）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_central_network)

### 资源/数据源依赖关系

```
huaweicloud_cc_central_network
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建云连接中心网络资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云连接中心网络资源：

```hcl
variable "central_network_name" {
  description = "The name of the central network"
  type        = string
}

variable "central_network_description" {
  description = "The description about the central network"
  type        = string
  default     = "Created by Terraform"
}

variable "enterprise_project_id" {
  description = "ID of the enterprise project that the central network belongs to"
  type        = string
  default     = "0"
}

variable "central_network_tags" {
  description = "The tags of the central network"
  type        = map(string)
  default     = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云连接中心网络资源
resource "huaweicloud_cc_central_network" "test" {
  name                  = var.central_network_name
  description           = var.central_network_description
  enterprise_project_id = var.enterprise_project_id

  tags = var.central_network_tags
}
```

**参数说明**：
- **name**：中心网络名称，通过引用输入变量central_network_name进行赋值
- **description**：中心网络的描述信息，通过引用输入变量central_network_description进行赋值，默认值为"Created by Terraform"
- **enterprise_project_id**：中心网络所属的企业项目ID，通过引用输入变量enterprise_project_id进行赋值，默认值为"0"
- **tags**：中心网络的标签，通过引用输入变量central_network_tags进行赋值，默认值包含"Owner"和"Env"标签

### 3. 预设资源部署所需的入参（可选）

本实践中，资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 云连接中心网络配置
central_network_name  = "tf_test_central_network"
enterprise_project_id = "0"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="central_network_name=test-network" -var="enterprise_project_id=0"`
2. 环境变量：`export TF_VAR_central_network_name=test-network` 和 `export TF_VAR_enterprise_project_id=0`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建云连接中心网络：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建云连接中心网络
4. 运行 `terraform show` 查看已创建的云连接中心网络详情

## 参考信息

- [华为云云连接产品文档](https://support.huaweicloud.com/cc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [中心网络最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cc/central-network)
