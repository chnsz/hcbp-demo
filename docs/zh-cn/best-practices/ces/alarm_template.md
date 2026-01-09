# 部署告警模板

## 应用场景

云监控服务（Cloud Eye Service，CES）告警模板是CES服务提供的告警规则模板功能，用于快速创建和管理告警规则。通过配置告警模板，您可以定义统一的告警策略，包括监控指标、告警阈值、告警级别等，然后基于模板快速创建多个告警规则，提高告警配置的效率和一致性。通过Terraform自动化创建CES告警模板，可以确保告警配置的规范化和标准化，简化运维管理。本最佳实践将介绍如何使用Terraform自动化创建CES告警模板。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [CES告警模板资源（huaweicloud_ces_alarm_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarm_template)

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建CES告警模板资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CES告警模板资源：

```hcl
variable "alarm_template_name" {
  description = "The name of the alarm template"
  type        = string
}

variable "alarm_template_description" {
  description = "The description of the alarm template"
  type        = string
  default     = ""
}

variable "alarm_template_policies" {
  description = "The policy list of the CES alarm template"
  type = list(object({
    namespace           = string
    metric_name         = string
    period              = number
    filter              = string
    comparison_operator = string
    count               = number
    suppress_duration   = number
    value               = number
    alarm_level         = number
    unit                = string
    dimension_name      = string
    hierarchical_value = list(object({
      critical = number
      major    = number
      minor    = number
      info     = number
    }))
  }))
  default = []
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CES告警模板资源
resource "huaweicloud_ces_alarm_template" "test" {
  name        = var.alarm_template_name
  description = var.alarm_template_description

  dynamic "policies" {
    for_each = var.alarm_template_policies

    content {
      namespace           = policies.value.namespace
      metric_name         = policies.value.metric_name
      period              = policies.value.period
      filter              = policies.value.filter
      comparison_operator = policies.value.comparison_operator
      count               = policies.value.count
      suppress_duration   = policies.value.suppress_duration
      value               = policies.value.value
      alarm_level         = policies.value.alarm_level
      unit                = policies.value.unit
      dimension_name      = policies.value.dimension_name
      hierarchical_value {
        critical = policies.value.hierarchical_value[0].critical
        major    = policies.value.hierarchical_value[0].major
        minor    = policies.value.hierarchical_value[0].minor
        info     = policies.value.hierarchical_value[0].info
      }
    }
  }
}
```

**参数说明**：
- **name**：告警模板名称，通过引用输入变量alarm_template_name进行赋值
- **description**：告警模板描述，通过引用输入变量alarm_template_description进行赋值，可选参数，默认值为空字符串
- **policies**：告警策略列表，通过引用输入变量alarm_template_policies进行赋值，每个策略包含以下参数：
  - **namespace**：服务命名空间，如SYS.APIG、SYS.ECS等
  - **metric_name**：告警指标名称
  - **period**：告警条件判断周期，单位为分钟
  - **filter**：数据聚合方式，如average（平均值）、max（最大值）、min（最小值）等
  - **comparison_operator**：告警阈值的比较条件，如>、>=、<、<=、=等
  - **count**：连续触发告警的次数
  - **suppress_duration**：告警抑制周期，单位为秒
  - **value**：告警阈值
  - **alarm_level**：告警级别，1-4分别表示紧急、重要、次要、提示
  - **unit**：告警阈值的单位字符串
  - **dimension_name**：资源维度名称
  - **hierarchical_value**：多级告警阈值配置，包含以下字段：
    - **critical**：紧急级别的阈值
    - **major**：重要级别的阈值
    - **minor**：次要级别的阈值
    - **info**：提示级别的阈值

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 告警模板配置
alarm_template_name        = "tf_test_alarm_template"
alarm_template_description = "Test alarm template for APIG service"

# 告警策略配置
alarm_template_policies = [
  {
    namespace           = "SYS.APIG"
    dimension_name      = "api_id"
    metric_name         = "req_count_2xx"
    period              = 1
    filter              = "average"
    comparison_operator = ">"
    value               = 10
    unit                = "times/minute"
    count               = 3
    alarm_level         = 2
    suppress_duration   = 300
    hierarchical_value = [
      {
        critical = 12
        major    = 10
        minor    = 8
        info     = 4
      }
    ]
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="alarm_template_name=my_template" -var="alarm_template_description=My description"`
2. 环境变量：`export TF_VAR_alarm_template_name=my_template` 和 `export TF_VAR_alarm_template_description=My description`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建CES告警模板：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建告警模板
4. 运行 `terraform show` 查看已创建的告警模板详情

> 注意：告警模板创建后，可以基于该模板快速创建多个告警规则。告警策略中的hierarchical_value用于配置多级告警阈值，可以根据不同的告警级别设置不同的阈值。告警级别1-4分别表示紧急、重要、次要、提示，可以根据业务需求选择合适的告警级别。

## 参考信息

- [华为云CES产品文档](https://support.huaweicloud.com/ces/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [告警模板最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ces/alarm-template)
