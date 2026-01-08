# 部署无状态负载

## 应用场景

云容器实例（Cloud Container Instance，CCI）无状态负载是CCI服务提供的容器应用部署方式，用于部署和管理无状态的容器应用。通过创建无状态负载，您可以实现容器的自动扩缩容、滚动更新和故障恢复等功能，提高应用的可用性和可靠性。通过Terraform自动化创建无状态负载，可以确保容器应用部署的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建CCI无状态负载，包括命名空间的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [CCI命名空间资源（huaweicloud_cciv2_namespace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_namespace)
- [CCI无状态负载资源（huaweicloud_cciv2_deployment）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_deployment)

### 资源/数据源依赖关系

```
huaweicloud_cciv2_namespace
    └── huaweicloud_cciv2_deployment
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建CCI命名空间资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CCI命名空间资源：

```hcl
variable "namespace_name" {
  description = "The name of CCI namespace"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCI命名空间资源
resource "huaweicloud_cciv2_namespace" "test" {
  name = var.namespace_name
}
```

**参数说明**：
- **name**：命名空间名称，通过引用输入变量namespace_name进行赋值

### 3. 创建CCI无状态负载资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CCI无状态负载资源：

```hcl
variable "deployment_name" {
  description = "The name of CCI deployment"
  type        = string
}

variable "instance_type" {
  description = "The instance type of CCI pod"
  type        = string
  default     = "general-computing"
}

variable "container_name" {
  description = "The name of container"
  type        = string
  default     = "c1"
}

variable "container_image" {
  description = "The image of container"
  type        = string
  default     = "alpine:latest"
}

variable "cpu_limit" {
  description = "The CPU limit of container"
  type        = string
  default     = "1"
}

variable "memory_limit" {
  description = "The memory limit of container"
  type        = string
  default     = "2G"
}

variable "image_pull_secret_name" {
  description = "The name of image pull secret"
  type        = string
  default     = "imagepull-secret"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCI无状态负载资源
resource "huaweicloud_cciv2_deployment" "test" {
  namespace = huaweicloud_cciv2_namespace.test.name
  name      = var.deployment_name

  selector {
    match_labels = {
      app = "template1"
    }
  }

  template {
    metadata {
      labels = {
        app = "template1"
      }

      annotations = {
        "resource.cci.io/instance-type" = var.instance_type
      }
    }

    spec {
      containers {
        name  = var.container_name
        image = var.container_image

        resources {
          limits = {
            cpu    = var.cpu_limit
            memory = var.memory_limit
          }

          requests = {
            cpu    = var.cpu_limit
            memory = var.memory_limit
          }
        }
      }

      image_pull_secrets {
        name = var.image_pull_secret_name
      }
    }
  }

  lifecycle {
    ignore_changes = [
      template.0.metadata.0.annotations,
    ]
  }
}
```

**参数说明**：
- **namespace**：命名空间名称，引用前面创建的CCI命名空间资源（huaweicloud_cciv2_namespace.test）的名称
- **name**：无状态负载名称，通过引用输入变量deployment_name进行赋值
- **selector.match_labels**：选择器匹配标签，用于匹配Pod，设置为app = "template1"
- **template.metadata.labels**：Pod模板标签，必须与选择器标签匹配，设置为app = "template1"
- **template.metadata.annotations**：Pod模板注解，通过resource.cci.io/instance-type注解指定实例类型，引用输入变量instance_type进行赋值，默认值为"general-computing"
- **template.spec.containers.name**：容器名称，通过引用输入变量container_name进行赋值，默认值为"c1"
- **template.spec.containers.image**：容器镜像，通过引用输入变量container_image进行赋值，默认值为"alpine:latest"
- **template.spec.containers.resources.limits.cpu**：CPU限制，通过引用输入变量cpu_limit进行赋值，默认值为"1"
- **template.spec.containers.resources.limits.memory**：内存限制，通过引用输入变量memory_limit进行赋值，默认值为"2G"
- **template.spec.containers.resources.requests.cpu**：CPU请求，通过引用输入变量cpu_limit进行赋值，默认值为"1"
- **template.spec.containers.resources.requests.memory**：内存请求，通过引用输入变量memory_limit进行赋值，默认值为"2G"
- **template.spec.image_pull_secrets.name**：镜像拉取密钥名称，通过引用输入变量image_pull_secret_name进行赋值，默认值为"imagepull-secret"

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# CCI无状态负载配置
deployment_name = "tf-test-deployment"
namespace_name  = "tf-test-namespace"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="deployment_name=test-deployment" -var="namespace_name=test-namespace"`
2. 环境变量：`export TF_VAR_deployment_name=test-deployment` 和 `export TF_VAR_namespace_name=test-namespace`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建CCI无状态负载：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CCI无状态负载
4. 运行 `terraform show` 查看已创建的CCI无状态负载详情

> 注意：无状态负载必须创建在已存在的命名空间中。选择器标签必须与Pod模板标签匹配。如果需要使用镜像拉取密钥，请提前创建。

## 参考信息

- [华为云CCI产品文档](https://support.huaweicloud.com/cci/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [无状态负载最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cci/deployment)
