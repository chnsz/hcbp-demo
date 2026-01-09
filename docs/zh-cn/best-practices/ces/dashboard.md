# 部署仪表板

## 应用场景

云监控服务（Cloud Eye Service，CES）仪表板是CES服务提供的监控数据可视化功能，用于集中展示多个监控指标和资源状态。通过配置仪表板，您可以创建自定义的监控视图，将多个监控指标以图表、表格等形式集中展示，方便运维人员快速了解云资源的整体运行状态。通过Terraform自动化创建CES仪表板，可以确保监控视图配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建CES仪表板。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [CES仪表板资源（huaweicloud_ces_dashboard）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_dashboard)

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建CES仪表板资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CES仪表板资源：

```hcl
variable "dashboard_name" {
  description = "The name of the alarm dashboard"
  type        = string
}

variable "dashboard_row_widget_num" {
  description = "The monitoring view display mode"
  type        = number
}

variable "dashboard_extend_info" {
  description = "The information about the extension"
  type = list(object({
    filter                  = string
    period                  = string
    display_time            = number
    refresh_time            = number
    from                    = number
    to                      = number
    screen_color            = string
    enable_screen_auto_play = bool
    time_interval           = number
    enable_legend           = bool
    full_screen_widget_num  = number
  }))
  default = []
}

variable "dashboard_id" {
  description = "The copied dashboard ID"
  type        = string
  default     = null
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the dashboard"
  type        = string
  default     = null
}

variable "is_favorite" {
  description = "Whether the dashboard is favorite"
  type        = bool
  default     = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CES仪表板资源
resource "huaweicloud_ces_dashboard" "test" {
  name           = var.dashboard_name
  row_widget_num = var.dashboard_row_widget_num

  dynamic "extend_info" {
    for_each = length(var.dashboard_extend_info) > 0 ? var.dashboard_extend_info : []

    content {
      filter                  = extend_info.value.filter
      period                  = extend_info.value.period
      display_time            = extend_info.value.display_time
      refresh_time            = extend_info.value.refresh_time
      from                    = extend_info.value.from
      to                      = extend_info.value.to
      screen_color            = extend_info.value.screen_color
      enable_screen_auto_play = extend_info.value.enable_screen_auto_play
      time_interval           = extend_info.value.time_interval
      enable_legend           = extend_info.value.enable_legend
      full_screen_widget_num  = extend_info.value.full_screen_widget_num
    }
  }

  dashboard_id         = var.dashboard_id
  enterprise_project_id = var.enterprise_project_id
  is_favorite         = var.is_favorite
}
```

**参数说明**：
- **name**：仪表板名称，通过引用输入变量dashboard_name进行赋值
- **row_widget_num**：监控视图显示模式，通过引用输入变量dashboard_row_widget_num进行赋值
- **extend_info**：扩展信息列表，通过引用输入变量dashboard_extend_info进行赋值，可选参数，默认值为空列表，每个扩展信息包含以下参数：
  - **filter**：指标聚合方法
  - **period**：指标聚合周期
  - **display_time**：显示时间
  - **refresh_time**：刷新时间
  - **from**：开始时间
  - **to**：结束时间
  - **screen_color**：监控大屏背景颜色
  - **enable_screen_auto_play**：是否启用监控大屏自动切换
  - **time_interval**：监控大屏自动切换时间间隔
  - **enable_legend**：是否启用图例
  - **full_screen_widget_num**：大屏显示视图数量
- **dashboard_id**：复制的仪表板ID，通过引用输入变量dashboard_id进行赋值，可选参数，默认值为null
- **enterprise_project_id**：仪表板所属的企业项目ID，通过引用输入变量enterprise_project_id进行赋值，可选参数，默认值为null
- **is_favorite**：是否收藏仪表板，通过引用输入变量is_favorite进行赋值，可选参数，默认值为false

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 仪表板配置
dashboard_name        = "tf_test_ces_dashboard"
dashboard_row_widget_num = 2

# 扩展信息配置
dashboard_extend_info = [
  {
    filter                  = "average"
    period                  = "1"
    display_time            = 3600
    refresh_time            = 60
    from                    = 0
    to                      = 3600
    screen_color            = "#000000"
    enable_screen_auto_play = true
    time_interval           = 30
    enable_legend           = true
    full_screen_widget_num   = 4
  }
]

# 可选配置
# dashboard_id         = "existing_dashboard_id"
# enterprise_project_id = "enterprise_project_id"
# is_favorite          = true
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="dashboard_name=my_dashboard" -var="dashboard_row_widget_num=2"`
2. 环境变量：`export TF_VAR_dashboard_name=my_dashboard` 和 `export TF_VAR_dashboard_row_widget_num=2`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建CES仪表板：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建仪表板
4. 运行 `terraform show` 查看已创建的仪表板详情

> 注意：仪表板创建后，可以在CES控制台中进行进一步的配置，添加监控指标和图表。通过设置extend_info可以配置监控大屏的显示参数，包括自动切换、刷新时间等。仪表板支持收藏功能，方便快速访问常用的监控视图。

## 参考信息

- [华为云CES产品文档](https://support.huaweicloud.com/ces/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [仪表板最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ces/dashboard)
