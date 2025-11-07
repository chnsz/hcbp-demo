# 部署CoreDNS插件

## 应用场景

云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。CoreDNS插件是CCE提供的一个DNS服务插件，用于为集群中的Pod提供DNS解析服务。CoreDNS是Kubernetes集群中默认的DNS服务器，负责将服务名称解析为IP地址，实现服务发现功能。通过部署CoreDNS插件，可以为集群提供可靠的DNS服务，支持服务间的高效通信。本最佳实践将介绍如何使用Terraform自动化部署一个CCE CoreDNS插件，包括CCE集群和插件模板的查询，以及CCE插件的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [CCE集群列表查询数据源（data.huaweicloud_cce_clusters）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cce_clusters)
- [CCE插件模板查询数据源（data.huaweicloud_cce_addon_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cce_addon_template)

### 资源

- [CCE插件资源（huaweicloud_cce_addon）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_addon)

### 资源/数据源依赖关系

```
data.huaweicloud_cce_clusters
    └── data.huaweicloud_cce_addon_template
        └── huaweicloud_cce_addon

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
  default     = "coredns"
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
- **name**：插件模板的名称，通过引用输入变量 `addon_template_name` 进行赋值，默认为"coredns"
- **version**：插件模板的版本，通过引用输入变量 `addon_version` 进行赋值

> 注意：插件版本必须与集群版本兼容。例如，如果集群版本是v1.32，则插件版本应该选择与v1.32兼容的版本。

### 4. 创建CCE插件资源

在TF文件中添加以下脚本以告知Terraform创建CCE插件资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE插件资源，用于为集群提供DNS服务
resource "huaweicloud_cce_addon" "test" {
  cluster_id    = var.cluster_id != "" ? var.cluster_id : try(data.huaweicloud_cce_clusters.test[0].clusters[0].id, null)
  template_name = var.addon_template_name
  version       = var.addon_version

  values {
    basic_json  = jsonencode(jsondecode(data.huaweicloud_cce_addon_template.test.spec).basic)
    custom_json = jsonencode(jsondecode(data.huaweicloud_cce_addon_template.test.spec).parameters.custom)
    flavor_json = jsonencode(jsondecode(data.huaweicloud_cce_addon_template.test.spec).parameters.flavor1)
  }
}
```

**参数说明**：
- **cluster_id**：CCE集群ID，如果指定了集群ID则使用该值，否则引用CCE集群列表查询数据源的第一个集群ID进行赋值
- **template_name**：插件模板的名称，通过引用输入变量 `addon_template_name` 进行赋值，默认为"coredns"
- **version**：插件的版本，通过引用输入变量 `addon_version` 进行赋值
- **values**：插件配置值块
  - **basic_json**：基础配置JSON，从插件模板的spec中解析basic部分并编码为JSON字符串
  - **custom_json**：自定义配置JSON，从插件模板的spec中解析parameters.custom部分并编码为JSON字符串
  - **flavor_json**：规格配置JSON，从插件模板的spec中解析parameters.flavor1部分并编码为JSON字符串

> 注意：插件的values配置包含三个部分：basic_json（基础配置）、custom_json（自定义配置）和flavor_json（规格配置）。本示例直接从插件模板中获取所有配置，无需额外修改。如果需要自定义配置，可以在创建插件前修改插件模板的配置，或者通过locals变量进行配置合并。

### 5. 预设资源部署所需的入参

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
cluster_id    = "your_cce_cluster_id"
addon_version = "1.30.33" # 集群版本为v1.32时，插件版本应为与v1.32兼容的版本
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
   - `cluster_id`：替换为实际的CCE集群ID
   - `addon_version`：根据集群版本选择合适的插件版本（插件版本应与集群版本兼容）
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="cluster_id=my-cluster-id" -var="addon_version=1.30.33"`
2. 环境变量：`export TF_VAR_cluster_id=my-cluster-id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CoreDNS插件
4. 运行 `terraform show` 查看已创建的CoreDNS插件

## 参考信息

- [华为云CCE产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE CoreDNS插件最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cce/addon-coredns)
