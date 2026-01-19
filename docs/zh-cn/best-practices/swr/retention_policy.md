# 部署保留策略

## 应用场景

容器镜像服务（Software Repository for Container，SWR）是华为云提供的容器镜像托管服务，支持Docker镜像的存储、管理和分发，帮助您实现容器应用的快速部署和持续集成。镜像保留策略是SWR服务的重要功能，用于自动清理过期的镜像版本，控制镜像仓库的存储空间。通过配置保留策略，可以设置保留的镜像数量或保留天数，并支持通过标签选择器筛选需要保留的镜像，实现镜像的自动化管理和存储优化。本最佳实践将介绍如何使用Terraform自动化部署保留策略，包括创建组织、创建镜像仓库和配置镜像保留策略。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [组织资源（huaweicloud_swr_organization）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/swr_organization)
- [镜像仓库资源（huaweicloud_swr_repository）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/swr_repository)
- [镜像保留策略资源（huaweicloud_swr_image_retention_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/swr_image_retention_policy)

### 资源/数据源依赖关系

```text
huaweicloud_swr_organization
    └── huaweicloud_swr_repository
        └── huaweicloud_swr_image_retention_policy
```

> 注意：镜像保留策略必须属于某个组织的某个镜像仓库，因此需要先创建组织和镜像仓库。保留策略通过引用组织名称和仓库名称来关联到对应的镜像仓库。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建组织

在TF文件（如main.tf）中添加以下脚本以创建组织：

```hcl
variable "organization_name" {
  description = "The organization name"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建组织资源
resource "huaweicloud_swr_organization" "test" {
  name = var.organization_name
}
```

**参数说明**：
- **name**：组织名称，通过引用输入变量 `organization_name` 进行赋值

> 注意：组织名称在SWR服务中必须唯一，用于标识和管理容器镜像仓库。

### 3. 创建镜像仓库

在TF文件（如main.tf）中添加以下脚本以创建镜像仓库：

```hcl
variable "repository_name" {
  description = "The repository name"
  type        = string
}

variable "category" {
  description = "The category"
  type        = string
  default     = "linux"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建镜像仓库资源
resource "huaweicloud_swr_repository" "test" {
  organization = huaweicloud_swr_organization.test.name
  name         = var.repository_name
  category     = var.category
}
```

**参数说明**：
- **organization**：组织名称，通过引用组织资源的名称进行赋值
- **name**：镜像仓库名称，通过引用输入变量 `repository_name` 进行赋值
- **category**：仓库分类，通过引用输入变量 `category` 进行赋值，默认为 `linux`，可选参数

> 注意：镜像仓库名称在组织内必须唯一。创建镜像仓库后，可以在该仓库下配置保留策略。

### 4. 配置镜像保留策略

在TF文件（如main.tf）中添加以下脚本以配置镜像保留策略：

```hcl
variable "policy_type" {
  description = "The policy type"
  type        = string
}

variable "policy_number" {
  description = "The policy number"
  type        = number
}

variable "tag_selectors_configuration" {
  description = "The configuration of the tag selectors"
  type        = list(object({
    kind    = string
    pattern = number
  }))

  default = []
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建镜像保留策略资源
resource "huaweicloud_swr_image_retention_policy" "test" {
  organization = huaweicloud_swr_organization.test.name
  repository   = huaweicloud_swr_repository.test.name
  type         = var.policy_type
  number       = var.policy_number

  dynamic "tag_selectors" {
    for_each = var.tag_selectors_configuration

    content {
      kind    = tag_selectors.value.kind
      pattern = tag_selectors.value.pattern
    }
  }
}
```

**参数说明**：
- **organization**：组织名称，通过引用组织资源的名称进行赋值
- **repository**：镜像仓库名称，通过引用镜像仓库资源的名称进行赋值
- **type**：策略类型，通过引用输入变量 `policy_type` 进行赋值，支持 `date_rule`（按日期规则）和 `tag_rule`（按标签规则）
- **number**：保留数量或天数，通过引用输入变量 `policy_number` 进行赋值，根据策略类型不同，表示保留的镜像数量或保留天数
- **tag_selectors**：标签选择器列表，通过动态块 `dynamic "tag_selectors"` 根据输入变量 `tag_selectors_configuration` 创建多个标签选择器，可选参数
  - **kind**：选择器类型，支持 `label`（标签）等
  - **pattern**：匹配模式，用于筛选需要保留的镜像

> 注意：保留策略用于自动清理过期的镜像版本。策略类型为 `date_rule` 时，`number` 表示保留天数；策略类型为 `tag_rule` 时，`number` 表示保留的镜像数量。标签选择器用于筛选需要保留的镜像，可以根据标签模式进行精确匹配。

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 组织配置（必填）
organization_name = "tf_test_swr_organization_name"

# 镜像仓库配置（必填）
repository_name = "tf_test_swr_repository_name"

# 保留策略配置（必填）
policy_type = "date_rule"
policy_number = 30

# 标签选择器配置（可选）
tag_selectors_configuration = [
  {
    kind    = "label"
    pattern = 1.1
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="organization_name=tf_test_swr_organization_name" -var="repository_name=tf_test_swr_repository_name"`
2. 环境变量：`export TF_VAR_organization_name=tf_test_swr_organization_name`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建组织、镜像仓库和镜像保留策略
4. 运行 `terraform show` 查看已创建的镜像保留策略

## 参考信息

- [华为云SWR产品文档](https://support.huaweicloud.com/swr/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [SWR保留策略最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/swr/retention-policy)
