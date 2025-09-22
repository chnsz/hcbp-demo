# 部署脚本

## 应用场景

云运维中心（Cloud Operations Center, COC）是华为云提供的一站式运维管理平台，为企业提供统一的运维管理入口。云运维中心通过脚本管理、任务调度、监控告警等功能，帮助企业实现自动化运维和智能化管理，提升运维效率和质量。

脚本管理是COC服务的核心功能之一，支持创建、管理和执行各种运维脚本，包括Shell脚本、Python脚本、PowerShell脚本等。通过脚本管理，可以实现运维脚本的版本控制、参数管理、风险等级评估等功能，为后续的脚本执行和任务调度奠定基础。本最佳实践将介绍如何使用Terraform自动化部署COC脚本，包括脚本创建和参数配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

无

### 资源

- [COC脚本资源（huaweicloud_coc_script）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/coc_script)

### 资源/数据源依赖关系

```
huaweicloud_coc_script.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建COC脚本

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建COC脚本资源：

```hcl
variable "script_name" {
  description = "脚本名称"
  type        = string
}

variable "script_description" {
  description = "脚本描述"
  type        = string
}

variable "script_risk_level" {
  description = "脚本风险级别"
  type        = string
}

variable "script_version" {
  description = "脚本版本"
  type        = string
}

variable "script_type" {
  description = "脚本类型"
  type        = string
}

variable "script_content" {
  description = "脚本内容"
  type        = string
}

variable "script_parameters" {
  description = "脚本参数列表"
  type = list(object({
    name        = string
    value       = string
    description = string
    sensitive   = optional(bool)
  }))

  nullable = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建COC脚本资源
resource "huaweicloud_coc_script" "test" {
  name        = var.script_name
  description = var.script_description
  risk_level  = var.script_risk_level
  version     = var.script_version
  type        = var.script_type
  content     = var.script_content

  dynamic "parameters" {
    for_each = var.script_parameters

    content {
      name        = parameters.value.name
      value       = parameters.value.value
      description = parameters.value.description
      sensitive   = parameters.value.sensitive
    }
  }
}
```

**参数说明**：
- **name**：脚本名称，通过引用输入变量script_name进行赋值
- **description**：脚本描述，通过引用输入变量script_description进行赋值
- **risk_level**：脚本风险级别，通过引用输入变量script_risk_level进行赋值
- **version**：脚本版本，通过引用输入变量script_version进行赋值
- **type**：脚本类型，通过引用输入变量script_type进行赋值
- **content**：脚本内容，通过引用输入变量script_content进行赋值
- **parameters.name**：参数名称，通过引用脚本参数列表中的name字段进行赋值
- **parameters.value**：参数值，通过引用脚本参数列表中的value字段进行赋值
- **parameters.description**：参数描述，通过引用脚本参数列表中的description字段进行赋值
- **parameters.sensitive**：参数是否敏感，通过引用脚本参数列表中的sensitive字段进行赋值

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 脚本配置
script_name        = "tf_coc_script"
script_description = "Created by terraform script"
script_risk_level  = "LOW"
script_version     = "1.0.0"
script_type        = "SHELL"
script_content     = <<EOF
#! /bin/bash
echo "hello world!"
EOF

# 脚本参数配置
script_parameters = [
  {
    name        = "name"
    value       = "world"
    description = "the first parameter"
  },
  {
    name        = "company"
    value       = "Huawei"
    description = "the second parameter"
    sensitive   = true
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="script_name=my-script" -var="script_type=SHELL"`
2. 环境变量：`export TF_VAR_script_name=my-script`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建COC脚本
4. 运行 `terraform show` 查看已创建的COC脚本

## 参考信息

- [华为云COC产品文档](https://support.huaweicloud.com/coc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [COC最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/coc)
