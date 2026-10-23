# 部署资源栈及执行计划

## 应用场景

资源编排服务（Resource Formation Service，RFS）是华为云提供的基础设施即代码（IaC）服务，支持以模板方式定义和编排云上资源。在实际交付中，用户往往希望在真正创建资源之前，先预览模板将要带来的资源变更，以降低误操作风险。RFS 的执行计划（Execution Plan）正是为此设计：它基于内联的 Terraform 模板体生成资源变更预览，而不会真正创建任何云资源。

本最佳实践将介绍如何使用Terraform创建一个空的RFS资源栈，并基于内联Terraform模板体为其创建执行计划，从而在部署前预览资源变更。由于资源栈本身不携带模板，因此不会部署任何云资源，也无需进行委托授权；执行计划仅用于预览模板体所描述的资源变更。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [资源栈（huaweicloud_rfs_stack）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_stack)
- [执行计划（huaweicloud_rfs_execution_plan_v2）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_execution_plan_v2)

### 资源/数据源依赖关系

```
huaweicloud_rfs_stack
    └── huaweicloud_rfs_execution_plan_v2
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建资源栈

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建资源栈资源
variable "stack_name" {
  description = "The name of the RFS resource stack"
  type        = string
}

variable "stack_description" {
  description = "The description of the RFS resource stack"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_stack" "test" {
  name        = var.stack_name
  description = var.stack_description
}
```

**参数说明**：
- **name**：资源栈名称，通过引用输入变量 stack_name 进行赋值
- **description**：资源栈描述，通过引用输入变量 stack_description 进行赋值，默认为空字符串

### 3. 创建执行计划

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建执行计划资源
variable "execution_plan_name" {
  description = "The name of the RFS execution plan"
  type        = string
}

variable "execution_plan_description" {
  description = "The description of the RFS execution plan"
  type        = string
  default     = ""
}

variable "execution_plan_template_body" {
  description = "The Terraform template body used by the RFS execution plan"
  type        = string
}

resource "huaweicloud_rfs_execution_plan_v2" "test" {
  stack_name          = huaweicloud_rfs_stack.test.name
  stack_id            = huaweicloud_rfs_stack.test.id
  execution_plan_name = var.execution_plan_name
  description         = var.execution_plan_description
  template_body       = var.execution_plan_template_body
}
```

**参数说明**：
- **stack_name**：执行计划所属资源栈名称，引用上一步创建的资源栈的 name 属性进行赋值
- **stack_id**：执行计划所属资源栈ID，引用上一步创建的资源栈的 id 属性进行赋值
- **execution_plan_name**：执行计划名称，通过引用输入变量 execution_plan_name 进行赋值
- **description**：执行计划描述，通过引用输入变量 execution_plan_description 进行赋值，默认为空字符串
- **template_body**：执行计划所使用的Terraform模板体，通过引用输入变量 execution_plan_template_body 进行赋值。模板体中必须声明 `required_providers` 块，并将 provider 源指定为 `huawei.com/provider/huaweicloud`，否则执行计划引擎无法解析 provider 包，导致执行计划创建失败

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
stack_name          = "tf-test-stack"
execution_plan_name = "tf-test-plan"

execution_plan_template_body = <<-EOT
terraform {
  required_providers {
    huaweicloud = {
      source  = "huawei.com/provider/huaweicloud"
      version = ">= 1.41.0"
    }
  }
}

resource "huaweicloud_vpc" "test" {
  name = "tf-test-vpc"
  cidr = "192.168.0.0/16"
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = "tf-test-subnet"
  cidr       = "192.168.1.0/24"
  gateway_ip = "192.168.1.1"
}
EOT
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="stack_name=my-stack"`
2. 环境变量：`export TF_VAR_stack_name=my-stack`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建资源栈及执行计划
4. 运行 `terraform show` 查看已创建的资源栈及执行计划

## 参考信息

- [华为云资源编排服务（RFS）产品文档](https://support.huaweicloud.com/rfs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [RFS资源栈及执行计划最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rfs/stack-with-execution-plan)
