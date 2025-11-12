# 部署云应用策略组伸缩策略

## 应用场景

华为云云桌面（Workspace）是一种基于云计算的桌面虚拟化服务，为企业用户提供安全、便捷的云上办公解决方案。云应用策略组伸缩策略是Workspace服务中云应用功能的重要组成部分，用于为云应用服务器组配置自动伸缩策略，根据会话使用情况自动调整服务器实例数量，实现资源的弹性扩展和成本优化。

通过云应用策略组伸缩策略，企业可以根据实际业务负载自动调整云应用服务器组的实例数量，当会话使用率超过阈值时自动扩容，当会话空闲时间达到设定值时自动缩容。这种自动伸缩机制可以帮助企业实现资源的按需分配，提高资源利用率，降低运营成本，同时确保用户访问体验。本最佳实践将介绍如何使用Terraform自动化部署Workspace云应用策略组伸缩策略，包括云应用服务器组创建、云应用组创建、策略组配置和伸缩策略配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [Workspace服务查询数据源（data.huaweicloud_workspace_service）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_service)

### 资源

- [Workspace应用服务器组资源（huaweicloud_workspace_app_server_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_app_server_group)
- [Workspace应用组资源（huaweicloud_workspace_app_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_app_group)
- [Workspace应用策略组资源（huaweicloud_workspace_app_policy_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_app_policy_group)
- [Workspace应用服务器组伸缩策略资源（huaweicloud_workspace_app_server_group_scaling_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_app_server_group_scaling_policy)

### 资源/数据源依赖关系

```
data.huaweicloud_workspace_service.test
    └── huaweicloud_workspace_app_server_group.test
        ├── huaweicloud_workspace_app_group.test
        │   └── huaweicloud_workspace_app_policy_group.test
        └── huaweicloud_workspace_app_server_group_scaling_policy.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

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
variable "app_server_group_name" {
  description = "云应用服务器组的名称"
  type        = string
}

variable "app_server_group_app_type" {
  description = "云应用服务器组的应用类型"
  type        = string
  default     = "SESSION_DESKTOP_APP"
}

variable "app_server_group_os_type" {
  description = "云应用服务器组的操作系统类型"
  type        = string
  default     = "Windows"
}

variable "app_server_group_flavor_id" {
  description = "云应用服务器组的规格ID"
  type        = string
}

variable "app_server_group_image_id" {
  description = "云应用服务器组的镜像ID"
  type        = string
}

variable "app_server_group_image_product_id" {
  description = "云应用服务器组的镜像产品ID"
  type        = string
}

variable "app_server_group_system_disk_type" {
  description = "云应用服务器组的系统盘类型"
  type        = string
  default     = "SAS"
}

variable "app_server_group_system_disk_size" {
  description = "云应用服务器组的系统盘大小（GB）"
  type        = number
  default     = 80

  validation {
    condition     = var.app_server_group_system_disk_size >= 40 && var.app_server_group_system_disk_size <= 2048
    error_message = "系统盘大小必须在40到2048 GB之间。"
  }
}

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
- **name**：云应用服务器组的名称，通过引用输入变量 `app_server_group_name` 进行赋值
- **app_type**：应用类型，通过引用输入变量 `app_server_group_app_type` 进行赋值，默认为"SESSION_DESKTOP_APP"
- **os_type**：操作系统类型，通过引用输入变量 `app_server_group_os_type` 进行赋值，默认为"Windows"
- **flavor_id**：规格ID，通过引用输入变量 `app_server_group_flavor_id` 进行赋值
- **image_type**：镜像类型，固定设置为"gold"（黄金镜像）
- **image_id**：镜像ID，通过引用输入变量 `app_server_group_image_id` 进行赋值
- **image_product_id**：镜像产品ID，通过引用输入变量 `app_server_group_image_product_id` 进行赋值
- **vpc_id**：VPC ID，根据Workspace服务查询数据源（data.huaweicloud_workspace_service）的返回结果进行赋值
- **subnet_id**：子网ID，根据Workspace服务查询数据源（data.huaweicloud_workspace_service）的返回结果进行赋值
- **system_disk_type**：系统盘类型，通过引用输入变量 `app_server_group_system_disk_type` 进行赋值，默认为"SAS"
- **system_disk_size**：系统盘大小，通过引用输入变量 `app_server_group_system_disk_size` 进行赋值，默认为80GB
- **is_vdi**：是否为VDI模式，固定设置为true

### 4. 创建Workspace云应用组

在TF文件中添加以下脚本以告知Terraform创建云应用组资源：

```hcl
variable "app_group_name" {
  description = "云应用组的名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Workspace云应用组资源
resource "huaweicloud_workspace_app_group" "test" {
  depends_on = [huaweicloud_workspace_app_server_group.test]

  server_group_id = huaweicloud_workspace_app_server_group.test.id
  name            = var.app_group_name
  type            = "SESSION_DESKTOP_APP"
  description     = "Created APP group by Terraform"
}
```

**参数说明**：
- **depends_on**：显式依赖声明，确保云应用服务器组创建完成后再创建云应用组
- **server_group_id**：云应用服务器组的ID，引用前面创建的云应用服务器组资源的ID
- **name**：云应用组的名称，通过引用输入变量 `app_group_name` 进行赋值
- **type**：云应用组的类型，固定设置为"SESSION_DESKTOP_APP"表示会话桌面应用组
- **description**：云应用组的描述，固定设置为"Created APP group by Terraform"

### 5. 创建Workspace云应用策略组

在TF文件中添加以下脚本以告知Terraform创建云应用策略组资源：

```hcl
variable "policy_group_name" {
  description = "策略组的名称"
  type        = string
}

variable "policy_group_priority" {
  description = "策略组的优先级"
  type        = number
  default     = 1
}

variable "policy_group_description" {
  description = "策略组的描述"
  type        = string
  default     = "Created APP policy group by Terraform"
}

variable "target_type" {
  description = "策略组的目标类型"
  type        = string
  default     = "APPGROUP"

  validation {
    condition     = contains(["APPGROUP", "ALL"], var.target_type)
    error_message = "target_type必须是'APPGROUP'或'ALL'。"
  }
}

variable "automatic_reconnection_interval" {
  description = "自动重连间隔（分钟）"
  type        = number
  default     = 10

  validation {
    condition     = var.automatic_reconnection_interval >= 1 && var.automatic_reconnection_interval <= 60
    error_message = "自动重连间隔必须在1到60分钟之间。"
  }
}

variable "session_persistence_time" {
  description = "会话持久化时间（分钟）"
  type        = number
  default     = 120

  validation {
    condition     = var.session_persistence_time >= 1 && var.session_persistence_time <= 1440
    error_message = "会话持久化时间必须在1到1440分钟之间。"
  }
}

variable "forbid_screen_capture" {
  description = "是否禁止截屏"
  type        = bool
  default     = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Workspace云应用策略组资源
resource "huaweicloud_workspace_app_policy_group" "test" {
  depends_on = [huaweicloud_workspace_app_group.test]

  name        = var.policy_group_name
  priority    = var.policy_group_priority
  description = var.policy_group_description

  targets {
    id   = var.target_type == "APPGROUP" ? huaweicloud_workspace_app_group.test.id : "default-apply-all-targets"
    name = var.target_type == "APPGROUP" ? huaweicloud_workspace_app_group.test.name : "All-Targets"
    type = var.target_type
  }

  policies = jsonencode({
    "client": {
      "automatic_reconnection_interval": var.automatic_reconnection_interval,
      "session_persistence_time":        var.session_persistence_time,
      "forbid_screen_capture":           var.forbid_screen_capture
    }
  })
}
```

**参数说明**：
- **depends_on**：显式依赖声明，确保云应用组创建完成后再创建策略组
- **name**：策略组的名称，通过引用输入变量 `policy_group_name` 进行赋值
- **priority**：策略组的优先级，通过引用输入变量 `policy_group_priority` 进行赋值，默认为1，数值越小优先级越高
- **description**：策略组的描述，通过引用输入变量 `policy_group_description` 进行赋值，默认为"Created APP policy group by Terraform"
- **targets**：策略组的目标配置块
  - **id**：目标ID，如果目标类型为"APPGROUP"则使用云应用组的ID，否则使用"default-apply-all-targets"表示应用于所有目标
  - **name**：目标名称，如果目标类型为"APPGROUP"则使用云应用组的名称，否则使用"All-Targets"
  - **type**：目标类型，通过引用输入变量 `target_type` 进行赋值，默认为"APPGROUP"表示应用于指定应用组，"ALL"表示应用于所有目标
- **policies**：策略配置，使用jsonencode函数将策略配置编码为JSON字符串
  - **client.automatic_reconnection_interval**：客户端自动重连间隔（分钟），通过引用输入变量 `automatic_reconnection_interval` 进行赋值，默认为10分钟
  - **client.session_persistence_time**：会话持久化时间（分钟），通过引用输入变量 `session_persistence_time` 进行赋值，默认为120分钟
  - **client.forbid_screen_capture**：是否禁止截屏，通过引用输入变量 `forbid_screen_capture` 进行赋值，默认为true

### 6. 创建Workspace云应用服务器组伸缩策略

在TF文件中添加以下脚本以告知Terraform创建云应用服务器组伸缩策略资源：

```hcl
variable "max_scaling_amount" {
  description = "可扩容的最大实例数"
  type        = number

  validation {
    condition     = var.max_scaling_amount >= 1 && var.max_scaling_amount <= 100
    error_message = "可扩容的最大实例数必须在1到100之间。"
  }
}

variable "single_expansion_count" {
  description = "单次扩容的实例数"
  type        = number

  validation {
    condition     = var.single_expansion_count >= 1 && var.single_expansion_count <= 10
    error_message = "单次扩容的实例数必须在1到10之间。"
  }
}

variable "session_usage_threshold" {
  description = "会话使用率阈值（百分比）"
  type        = number
  default     = 80

  validation {
    condition     = var.session_usage_threshold >= 1 && var.session_usage_threshold <= 100
    error_message = "会话使用率阈值必须在1到100之间。"
  }
}

variable "shrink_after_session_idle_minutes" {
  description = "会话空闲后缩容等待时间（分钟）"
  type        = number
  default     = 30

  validation {
    condition     = var.shrink_after_session_idle_minutes >= 1 && var.shrink_after_session_idle_minutes <= 1440
    error_message = "会话空闲后缩容等待时间必须在1到1440分钟之间。"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Workspace云应用服务器组伸缩策略资源
resource "huaweicloud_workspace_app_server_group_scaling_policy" "test" {
  depends_on = [huaweicloud_workspace_app_server_group.test]

  server_group_id        = huaweicloud_workspace_app_server_group.test.id
  max_scaling_amount     = var.max_scaling_amount
  single_expansion_count = var.single_expansion_count

  scaling_policy_by_session {
    session_usage_threshold           = var.session_usage_threshold
    shrink_after_session_idle_minutes = var.shrink_after_session_idle_minutes
  }
}
```

**参数说明**：
- **depends_on**：显式依赖声明，确保云应用服务器组创建完成后再创建伸缩策略
- **server_group_id**：云应用服务器组的ID，引用前面创建的云应用服务器组资源的ID
- **max_scaling_amount**：可扩容的最大实例数，通过引用输入变量 `max_scaling_amount` 进行赋值，取值范围为1到100
- **single_expansion_count**：单次扩容的实例数，通过引用输入变量 `single_expansion_count` 进行赋值，取值范围为1到10
- **scaling_policy_by_session**：基于会话的伸缩策略配置块
  - **session_usage_threshold**：会话使用率阈值（百分比），通过引用输入变量 `session_usage_threshold` 进行赋值，默认为80%，当会话使用率超过此阈值时触发扩容
  - **shrink_after_session_idle_minutes**：会话空闲后缩容等待时间（分钟），通过引用输入变量 `shrink_after_session_idle_minutes` 进行赋值，默认为30分钟，当会话空闲时间达到此值时触发缩容

> 注意：伸缩策略基于会话使用情况进行自动调整。当会话使用率超过阈值时，系统会自动扩容实例；当会话空闲时间达到设定值时，系统会自动缩容实例。这样可以实现资源的弹性扩展和成本优化。

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 云应用服务器组配置
app_server_group_name             = "tf_test_server_group"
app_server_group_flavor_id        = "workspace.appstream.general.xlarge.4"
app_server_group_image_id         = "2ac7b1fb-b198-422b-a45f-61ea285cb6e7"
app_server_group_image_product_id = "OFFI886188719633408000"

# 云应用组配置
app_group_name = "tf_test_app_group"

# 策略组配置
policy_group_name = "tf_test_policy_group"

# 伸缩策略配置
max_scaling_amount     = 10
single_expansion_count = 2
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="app_server_group_name=my-server-group" -var="max_scaling_amount=10"`
2. 环境变量：`export TF_VAR_app_server_group_name=my-server-group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建云应用服务器组、云应用组、云应用策略组和伸缩策略
4. 运行 `terraform show` 查看已创建的云应用策略组伸缩策略详情

## 参考信息

- [华为云Workspace产品文档](https://support.huaweicloud.com/workspace/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Workspace云应用策略组伸缩策略最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/app/policy_group_scaling_policy)
