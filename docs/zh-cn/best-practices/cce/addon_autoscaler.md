# 部署AutoScaler插件

## 应用场景

云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。AutoScaler插件是CCE提供的一个自动扩缩容插件，用于根据工作负载的需求自动调整集群中的节点数量。通过部署AutoScaler插件，可以实现集群的自动扩缩容，提高资源利用率，降低运维成本。本最佳实践将介绍如何使用Terraform自动化部署一个CCE AutoScaler插件，包括CCE集群、插件模板和IAM项目的查询，以及CCE插件的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [CCE集群列表查询数据源（data.huaweicloud_cce_clusters）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cce_clusters)
- [CCE插件模板查询数据源（data.huaweicloud_cce_addon_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cce_addon_template)
- [IAM项目列表查询数据源（data.huaweicloud_identity_projects）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### 资源

- [CCE插件资源（huaweicloud_cce_addon）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_addon)

### 资源/数据源依赖关系

```
data.huaweicloud_cce_clusters
    └── data.huaweicloud_cce_addon_template
        └── huaweicloud_cce_addon

data.huaweicloud_identity_projects
    └── huaweicloud_cce_addon (通过locals合并到custom配置中)

data.huaweicloud_cce_addon_template
    └── huaweicloud_cce_addon
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询CCE集群信息（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询CCE集群信息（如果未指定集群ID）：

```hcl
variable "cluster_id" {
  description = "CCE集群的ID"
  type        = string
  default     = ""

  validation {
    condition     = var.cluster_id != "" || var.cluster_name != ""
    error_message = "必须提供cluster_id或cluster_name中的一个"
  }
}

variable "cluster_name" {
  description = "CCE集群的名称"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下符合条件的CCE集群信息，用于创建插件相关资源
data "huaweicloud_cce_clusters" "test" {
  count = var.cluster_id == "" ? 1 : 0

  name = var.cluster_name
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询CCE集群信息，仅当 `var.cluster_id` 为空时查询CCE集群信息
- **name**：CCE集群的名称，通过引用输入变量 `cluster_name` 进行赋值

### 3. 通过数据源查询CCE插件模板信息

在TF文件中添加以下脚本以告知Terraform查询CCE插件模板信息：

```hcl
variable "addon_template_name" {
  description = "CCE插件模板的名称"
  type        = string
  default     = "autoscaler"
}

variable "addon_version" {
  description = "CCE插件模板的版本"
  type        = string
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下符合条件的CCE插件模板信息，用于创建插件相关资源
data "huaweicloud_cce_addon_template" "test" {
  cluster_id = var.cluster_id != "" ? var.cluster_id : try(data.huaweicloud_cce_clusters.test[0].clusters[0].id, null)
  name       = var.addon_template_name
  version    = var.addon_version
}
```

**参数说明**：
- **cluster_id**：CCE集群ID，如果指定了集群ID则使用该值，否则引用CCE集群列表查询数据源的第一个集群ID进行赋值
- **name**：插件模板的名称，通过引用输入变量 `addon_template_name` 进行赋值，默认为"autoscaler"
- **version**：插件模板的版本，通过引用输入变量 `addon_version` 进行赋值

> 注意：插件版本必须与集群版本兼容。例如，如果集群版本是v1.32，则插件版本应该选择1.32.x系列。

### 4. 通过数据源查询IAM项目信息（可选）

在TF文件中添加以下脚本以告知Terraform查询IAM项目信息（如果未指定项目ID）：

```hcl
variable "project_id" {
  description = "项目的ID"
  type        = string
  default     = ""
}

variable "region_name" {
  description = "区域名称"
  type        = string
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下符合条件的IAM项目信息，用于创建插件相关资源
data "huaweicloud_identity_projects" "test" {
  count = var.project_id == "" ? 1 : 0

  name = var.region_name
}
```

**参数说明**：
- **count**：数据源的查询数，用于控制是否查询IAM项目信息，仅当 `var.project_id` 为空时查询IAM项目信息
- **name**：区域名称，通过引用输入变量 `region_name` 进行赋值，用于查询当前区域的项目信息

### 5. 配置插件参数

在TF文件中添加以下脚本以配置插件参数：

```hcl
# 本地变量，用于解析和合并插件模板的配置
locals {
  # 解析插件模板中的custom配置
  original_custom = jsondecode(data.huaweicloud_cce_addon_template.test.spec).parameters.custom

  # 合并custom配置，添加cluster_id和tenant_id字段，保留其他字段
  merged_custom = jsonencode({
    # 与模板相比，需要在custom中添加的字段：cluster_id和tenant_id
    cluster_id                         = var.cluster_id != "" ? var.cluster_id : try(data.huaweicloud_cce_clusters.test[0].clusters[0].id, null)
    tenant_id                          = var.project_id != "" ? var.project_id : try(data.huaweicloud_identity_projects.test[0].projects[0].id, "")
    # 其余custom字段的值全部保留
    annotations                        = local.original_custom.annotations
    coresTotal                         = local.original_custom.coresTotal
    expander                           = local.original_custom.expander
    ignoreDaemonSetsUtilization        = local.original_custom.ignoreDaemonSetsUtilization
    ignoreLocalVolumeNodeAffinity      = local.original_custom.ignoreLocalVolumeNodeAffinity
    initialNodeGroupBackoffDuration    = local.original_custom.initialNodeGroupBackoffDuration
    logLevel                           = local.original_custom.logLevel
    maxEmptyBulkDeleteFlag             = local.original_custom.maxEmptyBulkDeleteFlag
    maxNodeGroupBackoffDuration        = local.original_custom.maxNodeGroupBackoffDuration
    maxNodeGroupBinPackingDuration     = local.original_custom.maxNodeGroupBinPackingDuration
    maxNodeProvisionTime               = local.original_custom.maxNodeProvisionTime
    maxNodesTotal                      = local.original_custom.maxNodesTotal
    memoryTotal                        = local.original_custom.memoryTotal
    multiAZBalance                     = local.original_custom.multiAZBalance
    multiAZEnabled                     = local.original_custom.multiAZEnabled
    newEphemeralVolumesPodScaleUpDelay = local.original_custom.newEphemeralVolumesPodScaleUpDelay
    node_match_expressions             = local.original_custom.node_match_expressions
    podDisruptionBudget                = local.original_custom.podDisruptionBudget
    resetUnNeededWhenScaleUp           = local.original_custom.resetUnNeededWhenScaleUp
    scaleDownDelayAfterAdd             = local.original_custom.scaleDownDelayAfterAdd
    scaleDownDelayAfterDelete          = local.original_custom.scaleDownDelayAfterDelete
    scaleDownDelayAfterFailure         = local.original_custom.scaleDownDelayAfterFailure
    scaleDownEnabled                   = local.original_custom.scaleDownEnabled
    scaleDownUnneededTime              = local.original_custom.scaleDownUnneededTime
    scaleDownUtilizationThreshold      = local.original_custom.scaleDownUtilizationThreshold
    scaleUpCpuUtilizationThreshold     = local.original_custom.scaleUpCpuUtilizationThreshold
    scaleUpMemUtilizationThreshold     = local.original_custom.scaleUpMemUtilizationThreshold
    scaleUpUnscheduledPodEnabled       = local.original_custom.scaleUpUnscheduledPodEnabled
    scaleUpUtilizationEnabled          = local.original_custom.scaleUpUtilizationEnabled
    scanInterval                       = local.original_custom.scanInterval
    skipNodesWithCustomControllerPods  = local.original_custom.skipNodesWithCustomControllerPods
    tolerations                        = local.original_custom.tolerations
    unremovableNodeRecheckTimeout      = local.original_custom.unremovableNodeRecheckTimeout
  })
}
```

**参数说明**：
- **original_custom**：从插件模板中解析出的原始custom配置
- **merged_custom**：合并后的custom配置，包含：
  - **cluster_id**：CCE集群ID，如果指定了集群ID则使用该值，否则引用CCE集群列表查询数据源的第一个集群ID
  - **tenant_id**：项目ID，如果指定了项目ID则使用该值，否则引用IAM项目列表查询数据源的第一个项目ID
  - 其他字段：保留插件模板中的原始配置值

> 注意：插件模板的custom配置中包含了AutoScaler的各种参数，如扩缩容阈值、延迟时间、节点匹配表达式等。本示例保留了模板中的所有原始配置，仅添加了必需的cluster_id和tenant_id字段。如果需要自定义这些参数，可以在locals中修改对应的字段值。

### 6. 创建CCE插件资源

在TF文件中添加以下脚本以告知Terraform创建CCE插件资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE插件资源，用于为集群提供自动扩缩容功能
resource "huaweicloud_cce_addon" "test" {
  cluster_id    = var.cluster_id != "" ? var.cluster_id : try(data.huaweicloud_cce_clusters.test[0].clusters[0].id, null)
  template_name = var.addon_template_name
  version       = var.addon_version

  values {
    basic_json  = jsonencode(jsondecode(data.huaweicloud_cce_addon_template.test.spec).basic)
    custom_json = local.merged_custom
    flavor_json = jsonencode(jsondecode(data.huaweicloud_cce_addon_template.test.spec).parameters.flavor1)
  }
}
```

**参数说明**：
- **cluster_id**：CCE集群ID，如果指定了集群ID则使用该值，否则引用CCE集群列表查询数据源的第一个集群ID进行赋值
- **template_name**：插件模板的名称，通过引用输入变量 `addon_template_name` 进行赋值，默认为"autoscaler"
- **version**：插件的版本，通过引用输入变量 `addon_version` 进行赋值
- **values**：插件配置值块
  - **basic_json**：基础配置JSON，从插件模板的spec中解析basic部分并编码为JSON字符串
  - **custom_json**：自定义配置JSON，使用本地变量 `merged_custom` 的值（包含cluster_id和tenant_id）
  - **flavor_json**：规格配置JSON，从插件模板的spec中解析parameters.flavor1部分并编码为JSON字符串

> 注意：插件的values配置包含三个部分：basic_json（基础配置）、custom_json（自定义配置）和flavor_json（规格配置）。本示例从插件模板中获取基础配置和规格配置，并合并自定义配置（添加了cluster_id和tenant_id）。

### 7. 预设资源部署所需的入参

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
cluster_id    = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
addon_version = "1.32.5" # 集群版本为v1.32时，插件版本应为1.32.x系列
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
   - `cluster_id`：替换为实际的CCE集群ID
   - `addon_version`：根据集群版本选择合适的插件版本（插件版本应与集群版本兼容）
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="cluster_id=my-cluster-id" -var="addon_version=1.32.5"`
2. 环境变量：`export TF_VAR_cluster_id=my-cluster-id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建AutoScaler插件
4. 运行 `terraform show` 查看已创建的AutoScaler插件

## 参考信息

- [华为云CCE产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE AutoScaler插件最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cce/addon-autoscaler)
