# 部署带命名空间的网格

## 应用场景

应用服务网格（Application Service Mesh，简称ASM）提供非侵入式的微服务治理解决方案，支持完整的生命周期管理和流量治理，兼容Kubernetes和Istio生态，功能包括负载均衡、熔断、故障注入等多种治理能力，并内置金丝雀、蓝绿灰度发布流程，提供一站式自动化的发布管理。ASM深度对接华为云云容器引擎（CCE），可为客户提供开箱即用的服务网格体验。

本最佳实践将介绍如何使用Terraform自动化部署一个配置命名空间Sidecar注入的ASM网格，包括按需创建CCE命名空间、将网格组件安装到指定CCE节点，以及配置指定命名空间的Sidecar注入。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [CCE命名空间资源（huaweicloud_cce_namespace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_namespace)
- [ASM网格资源（huaweicloud_asm_mesh）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/asm_mesh)

### 资源/数据源依赖关系

```
huaweicloud_cce_namespace
    └── huaweicloud_asm_mesh
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建CCE命名空间

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CCE命名空间资源，用于ASM网格的Sidecar注入：

```hcl
variable "namespaces" {
  description = "The list of existing namespace names for sidecar injection"
  type        = list(string)
  default     = []
}

variable "cluster_id" {
  description = "The ID of the CCE cluster to be associated with the ASM mesh"
  type        = string
}

variable "namespace_name" {
  description = "The name of the namespace to create"
  type        = string
  default     = ""

  validation {
    condition     = length(var.namespaces) != 0 || var.namespace_name != ""
    error_message = "The 'namespace_name' is required when 'namespaces' is not specified."
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE命名空间资源，用于ASM网格的Sidecar注入
resource "huaweicloud_cce_namespace" "test" {
  count = length(var.namespaces) == 0 ? 1 : 0

  cluster_id = var.cluster_id
  name       = var.namespace_name
}
```

**参数说明**：
- **count**：命名空间资源的创建数，用于控制是否创建命名空间，仅当`var.namespaces`为空时创建命名空间资源
- **cluster_id**：命名空间所属CCE集群的ID，通过引用输入变量cluster_id进行赋值
- **name**：命名空间名称，通过引用输入变量namespace_name进行赋值

> 注意：当已通过`namespaces`指定现有命名空间时，不会创建新的CCE命名空间；未指定时必须提供`namespace_name`。

### 3. 创建ASM网格

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ASM网格资源：

```hcl
variable "mesh_name" {
  description = "The name of the ASM mesh"
  type        = string
}

variable "mesh_type" {
  description = "The type of the ASM mesh"
  type        = string
  default     = "InCluster"
  nullable    = false
}

variable "mesh_version" {
  description = "The version of the ASM mesh"
  type        = string
}

variable "tags" {
  description = "The key/value pairs to associate with the ASM mesh"
  type        = map(string)
  default     = {}
}

variable "node_id" {
  description = "The ID of the node where ASM mesh components will be installed"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ASM网格资源
resource "huaweicloud_asm_mesh" "test" {
  name    = var.mesh_name
  type    = var.mesh_type
  version = var.mesh_version
  tags    = var.tags

  extend_params {
    clusters {
      cluster_id = var.cluster_id

      installation {
        nodes {
          field_selector {
            key      = "UID"
            operator = "In"
            values   = [var.node_id]
          }
        }
      }

      injection {
        namespaces {
          field_selector {
            key      = "Name"
            operator = "In"
            values   = length(var.namespaces) > 0 ? var.namespaces : [huaweicloud_cce_namespace.test[0].name]
          }
        }
      }
    }
  }
}
```

**参数说明**：
- **name**：ASM网格的名称，通过引用输入变量mesh_name进行赋值
- **type**：ASM网格的类型，通过引用输入变量mesh_type进行赋值，默认为"InCluster"
- **version**：ASM网格的版本，通过引用输入变量mesh_version进行赋值
- **tags**：关联到ASM网格的标签键值对，通过引用输入变量tags进行赋值，默认为空映射
- **extend_params**：ASM网格的扩展参数配置块
  - **clusters**：网格中的集群信息配置块
    - **cluster_id**：关联的CCE集群ID，通过引用输入变量cluster_id进行赋值
    - **installation**：网格组件安装配置块
      - **nodes**：网格组件安装节点配置块
        - **field_selector**：字段选择器配置块
          - **key**：选择器的键，固定为"UID"
          - **operator**：选择器的操作符，固定为"In"
          - **values**：选择器的值列表，通过引用输入变量node_id进行赋值，用于指定安装网格组件的节点
    - **injection**：Sidecar注入配置块
      - **namespaces**：Sidecar注入命名空间配置块
        - **field_selector**：字段选择器配置块
          - **key**：选择器的键，固定为"Name"
          - **operator**：选择器的操作符，固定为"In"
          - **values**：选择器的值列表，当`var.namespaces`不为空时使用该列表，否则引用前面创建的CCE命名空间资源（huaweicloud_cce_namespace.test）的名称

> 注意：创建带命名空间配置的ASM网格前，请确保目标CCE集群及节点已就绪，并将`cluster_id`、`node_id`替换为实际可用的资源ID。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# ASM网格配置
mesh_name    = "with-namespace-mesh"
mesh_version = "1.18.7-r7"

# CCE集群、节点与命名空间
cluster_id = "your-cce-cluster-id"
node_id    = "your-cce-node-id"

namespace_name = "your-cce-namespace-name"

# 网格标签
tags = {
  foo = "bar"
  key = "value"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="mesh_name=with-namespace-mesh" -var="mesh_version=1.18.7-r7"`
2. 环境变量：`export TF_VAR_mesh_name=with-namespace-mesh`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建带命名空间的ASM网格
4. 运行 `terraform show` 查看已创建的带命名空间的ASM网格

## 参考信息

- [华为云ASM产品文档](https://support.huaweicloud.com/asm/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ASM带命名空间网格最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/asm/with-namespace)
