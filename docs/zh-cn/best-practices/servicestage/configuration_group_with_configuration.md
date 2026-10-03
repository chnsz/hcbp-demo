# 部署配置组与配置文件

## 应用场景

ServiceStage是华为云提供的一站式应用管理服务，支持应用的开发、构建、部署、治理与运维全生命周期管理。在微服务与云原生应用场景中，配置管理是应用部署与运行的关键环节，通过配置组与配置文件可以集中管理应用的环境变量、启动参数与业务配置，实现配置与代码分离。

本最佳实践将介绍如何使用Terraform自动化创建ServiceStage配置组及其下属的配置文件，帮助您以基础设施即代码（IaC）的方式高效管理应用配置，为后续的应用部署与配置更新奠定基础。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [ServiceStage配置组（huaweicloud_servicestagev3_configuration_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_configuration_group)
- [ServiceStage配置文件（huaweicloud_servicestagev3_configuration）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_configuration)

### 资源/数据源依赖关系

```
huaweicloud_servicestagev3_configuration_group
    └── huaweicloud_servicestagev3_configuration
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建ServiceStage配置组

在TF文件（如main.tf）中添加以下脚本以创建配置组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ServiceStage配置组
variable "configuration_group_name" {
  description = "The name of the configuration group"
  type        = string
}

variable "configuration_group_description" {
  description = "The description of the configuration group"
  type        = string
  default     = ""
}

resource "huaweicloud_servicestagev3_configuration_group" "test" {
  name        = var.configuration_group_name
  description = var.configuration_group_description
}
```

**参数说明**：
- **name**：配置组名称，通过引用输入变量 configuration_group_name 进行赋值，名称长度为2到64个字符，仅允许字母、数字、连字符（-）和下划线（_），且必须以字母开头、以字母或数字结尾
- **description**：配置组描述，通过引用输入变量 configuration_group_description 进行赋值，该参数为可选参数

### 3. 创建ServiceStage配置文件

在TF文件（如main.tf）中添加以下脚本以创建属于上述配置组的配置文件：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ServiceStage配置文件
variable "configuration_name" {
  description = "The name of the configuration file"
  type        = string
}

variable "configuration_content" {
  description = "The content of the configuration file"
  type        = string
}

variable "configuration_description" {
  description = "The description of the configuration file"
  type        = string
  default     = ""
}

resource "huaweicloud_servicestagev3_configuration" "test" {
  config_group_id = huaweicloud_servicestagev3_configuration_group.test.id
  name            = var.configuration_name
  type            = "properties"
  content         = var.configuration_content
  description     = var.configuration_description

  lifecycle {
    ignore_changes = [
      type
    ]
  }
}
```

**参数说明**：
- **config_group_id**：配置文件所属的配置组ID，通过引用资源 huaweicloud_servicestagev3_configuration_group.test.id 进行赋值
- **name**：配置文件名称，通过引用输入变量 configuration_name 进行赋值，命名规则与配置组名称一致
- **type**：配置文件类型，当前仅支持 `yaml` 和 `properties`，本实践使用 `properties`，需确保 content 内容与声明的类型匹配
- **content**：配置文件内容，通过引用输入变量 configuration_content 进行赋值，支持使用 ServiceStage 系统变量（通过 `$${VARIABLE_NAME}` 引用）
- **description**：配置文件描述，通过引用输入变量 configuration_description 进行赋值，该参数为可选参数
- **lifecycle.ignore_changes**：由于查询详情接口不返回 type 属性，此处对 type 参数忽略变更，避免产生永久性的就地更新差异；如需修改 type，请先将其从 ignore_changes 中移除

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
configuration_group_name        = "tf_test_ss_config_group"
configuration_group_description = "Created by Terraform for ServiceStage best practice example"
configuration_name              = "tf_test_ss_configuration"
configuration_content           = "spring.application.name = tf-example-service"
configuration_description       = "Created by Terraform for ServiceStage best practice example"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="configuration_group_name=my-config-group"`
2. 环境变量：`export TF_VAR_configuration_group_name=my-config-group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建ServiceStage配置组与配置文件
4. 运行 `terraform show` 查看已创建的ServiceStage配置组与配置文件

## 参考信息

- [华为云ServiceStage产品文档](https://support.huaweicloud.com/servicestage/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ServiceStage配置组与配置文件最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/servicestage/configuration-group-with-configuration)
