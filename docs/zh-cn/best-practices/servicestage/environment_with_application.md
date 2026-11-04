# 部署环境与应用及其应用配置

## 应用场景

应用管理与运维平台（ServiceStage）是华为云提供的一站式应用托管平台，面向企业应用与云原生应用，提供应用开发、构建、部署、运维的全生命周期管理能力。在将应用迁移上云或构建新的云原生应用时，通常需要先准备承载应用的网络环境，再创建应用运行环境与应用，并为应用配置环境变量等运行参数。

本最佳实践将介绍如何使用Terraform自动化创建VPC、绑定该VPC的ServiceStage环境、应用以及应用级环境变量配置，帮助您以基础设施即代码（IaC）的方式快速搭建应用托管的基础环境，并为后续的组件部署与运维工作奠定基础。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [ServiceStage环境（huaweicloud_servicestagev3_environment）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_environment)
- [ServiceStage应用（huaweicloud_servicestagev3_application）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_application)
- [ServiceStage应用配置（huaweicloud_servicestagev3_application_configuration）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_application_configuration)

### 资源/数据源依赖关系

```
huaweicloud_vpc
    └── huaweicloud_servicestagev3_environment
            └── huaweicloud_servicestagev3_application_configuration
huaweicloud_servicestagev3_application
    └── huaweicloud_servicestagev3_application_configuration
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云资源
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = "192.168.0.0/16"
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，此处固定为 `192.168.0.0/16`

### 3. 创建ServiceStage环境

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ServiceStage环境资源
variable "environment_name" {
  description = "The name of the environment"
  type        = string
}

variable "environment_description" {
  description = "The description of the environment"
  type        = string
  default     = ""
}

resource "huaweicloud_servicestagev3_environment" "test" {
  name        = var.environment_name
  description = var.environment_description
  vpc_id      = huaweicloud_vpc.test.id
}
```

**参数说明**：
- **name**：环境名称，通过引用输入变量 environment_name 进行赋值，长度需为2到64个字符，仅允许字母、数字、中划线（-）和下划线（_），且必须以字母开头、以字母或数字结尾
- **description**：环境描述，通过引用输入变量 environment_description 进行赋值，长度不能超过128个字符
- **vpc_id**：环境绑定的VPC ID，引用上一步创建的VPC的ID进行赋值；该参数不支持更新，修改后会重建环境

### 4. 创建ServiceStage应用

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ServiceStage应用资源
variable "application_name" {
  description = "The name of the application"
  type        = string
}

variable "application_description" {
  description = "The description of the application"
  type        = string
  default     = ""
}

resource "huaweicloud_servicestagev3_application" "test" {
  name        = var.application_name
  description = var.application_description
}
```

**参数说明**：
- **name**：应用名称，通过引用输入变量 application_name 进行赋值，长度需为2到64个字符，仅允许字母、数字、中划线（-）和下划线（_），且必须以字母开头、以字母或数字结尾
- **description**：应用描述，通过引用输入变量 application_description 进行赋值，长度不能超过128个字符

### 5. 创建应用配置

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ServiceStage应用配置资源
variable "application_configuration_env_name" {
  description = "The name of the environment variable"
  type        = string
}

variable "application_configuration_env_value" {
  description = "The value of the environment variable"
  type        = string
}

resource "huaweicloud_servicestagev3_application_configuration" "test" {
  environment_id = huaweicloud_servicestagev3_environment.test.id
  application_id = huaweicloud_servicestagev3_application.test.id

  configuration {
    env {
      name  = var.application_configuration_env_name
      value = var.application_configuration_env_value
    }
  }
}
```

**参数说明**：
- **environment_id**：应用配置所属的环境ID，引用上一步创建的ServiceStage环境的ID进行赋值；该参数不支持更新，修改后会重建资源
- **application_id**：应用配置所属的应用ID，引用上一步创建的ServiceStage应用的ID进行赋值；该参数不支持更新，修改后会重建资源
- **configuration.env.name**：环境变量名称，通过引用输入变量 application_configuration_env_name 进行赋值，长度需为1到64个字符，且必须以字母、中划线（-）或下划线（_）开头
- **configuration.env.value**：环境变量值，通过引用输入变量 application_configuration_env_value 进行赋值

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name                            = "tf_test_ss_vpc"
environment_name                    = "tf_test_ss_environment"
environment_description             = "Created by Terraform for ServiceStage best practice example"
application_name                    = "tf_test_ss_application"
application_description             = "Created by Terraform for ServiceStage best practice example"
application_configuration_env_name  = "TF_EXAMPLE_APP_ENV"
application_configuration_env_value = "tf_example_value"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建ServiceStage环境、应用及应用配置
4. 运行 `terraform show` 查看已创建的ServiceStage环境、应用及应用配置

## 参考信息

- [华为云ServiceStage产品文档](https://support.huaweicloud.com/servicestage/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ServiceStage环境与应用及其应用配置最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/servicestage/environment-with-application)
