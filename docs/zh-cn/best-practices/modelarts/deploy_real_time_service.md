# 部署实时推理服务

## 应用场景

魔坊（ModelArts）在线推理服务支持将训练完成的模型部署为实时推理服务，提供低延迟、高可用的模型推理能力。实时推理服务（REAL_TIME）适用于对响应时延敏感的业务场景，支持多实例组部署、健康检查、滚动升级及日志采集等企业级特性。

本最佳实践适用于在ModelArts专属资源池上部署实时推理服务的场景，涵盖V2资源池查询、在线推理服务创建及实例组、单元配置、运行时配置、升级策略和日志配置等。本最佳实践将介绍如何使用Terraform自动化部署上述资源，实现实时推理服务的Infrastructure as Code管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [ModelArts V2资源池列表查询数据源（data.huaweicloud_modelartsv2_resource_pools）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelartsv2_resource_pools)

### 资源

- [ModelArts V2在线推理服务资源（huaweicloud_modelartsv2_service）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelartsv2_service)

### 资源/数据源依赖关系

```
data.huaweicloud_modelartsv2_resource_pools
    └── huaweicloud_modelartsv2_service
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询ModelArts V2资源池信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询ModelArts V2资源池信息，其查询结果用于获取专属资源池对应的工作空间ID及实例组pool_id：

```hcl
variable "service_group_pool_id" {
  type        = string
  description = "The ID of the dedicated resource pool for the instance group"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下ModelArts V2资源池信息，用于获取工作空间ID及实例组pool_id
data "huaweicloud_modelartsv2_resource_pools" "test" {}

locals {
  resource_pool = try([for pool in data.huaweicloud_modelartsv2_resource_pools.test.resource_pools : pool if pool.metadata[0].name ==
  var.service_group_pool_id][0], {})
}
```

**参数说明**：
- 此数据源无需额外参数，会自动查询当前区域的所有V2资源池
- **service_group_pool_id**：实例组所使用的专属资源池ID，通过引用输入变量service_group_pool_id进行赋值
- **resource_pool**：本地变量，根据service_group_pool_id匹配对应的V2资源池信息

### 3. 创建ModelArts V2实时推理服务

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts V2实时推理服务资源：

```hcl
variable "service_name" {
  type        = string
  description = "The name of the online inference service"
}

variable "service_version" {
  type        = string
  description = "The version of the online inference service"
}

variable "service_type" {
  type        = string
  default     = "REAL_TIME"
  description = "The type of the service"
}

variable "service_deploy_type" {
  type        = string
  default     = null
  description = "The deploy type of the service"
}

variable "service_description" {
  type        = string
  default     = ""
  description = "The description of the online inference service"
}

variable "service_group_framework" {
  type        = string
  default     = "COMMON"
  description = "The algorithm framework of the instance group"
}

variable "service_group_name" {
  type        = string
  description = "The name of the instance group"
}

variable "service_group_weight" {
  type        = number
  default     = 100
  description = "The weight percentage of the instance group"
}

variable "service_group_count" {
  type        = number
  default     = 1
  description = "The number of service instances in the deployment scenario"
}

variable "service_unit_configs" {
  type = list(object({
    image = object({
      source   = string
      swr_path = string
      id       = optional(string)
    })

    role = optional(string)

    custom_spec = optional(object({
      memory = number
      cpu    = optional(number)
      gpu    = optional(number)
      ascend = optional(number)
    }))

    flavor = optional(string)

    models = optional(list(object({
      source     = string
      mount_path = string
      address    = optional(string)
      source_id  = optional(string)
    })), [])

    codes = optional(list(object({
      source     = string
      mount_path = string
      address    = optional(string)
      source_id  = optional(string)
    })), [])

    count = optional(number)
    cmd   = optional(string)
    envs  = optional(map(string))

    readiness_health = optional(object({
      initial_delay_seconds = number
      timeout_seconds       = number
      period_seconds        = number
      failure_threshold     = number
      check_method          = string
      command               = optional(string)
      url                   = optional(string)
    }))

    startup_health = optional(object({
      initial_delay_seconds = number
      timeout_seconds       = number
      period_seconds        = number
      failure_threshold     = number
      check_method          = string
      command               = optional(string)
      url                   = optional(string)
    }))

    liveness_health = optional(object({
      initial_delay_seconds = number
      timeout_seconds       = number
      period_seconds        = number
      failure_threshold     = number
      check_method          = string
      command               = optional(string)
      url                   = optional(string)
    }))

    port     = optional(number)
    recovery = optional(string)
  }))

  description = "The unit configurations of the instance group"
}

variable "service_runtime_config" {
  type        = string
  description = "The runtime configuration of the service, in JSON format"
}

variable "service_upgrade_config" {
  type        = string
  description = "The upgrade configuration of the service, in JSON format"
}

variable "service_log_configs" {
  type = list(object({
    type          = string
    log_group_id  = optional(string)
    log_stream_id = optional(string)
  }))

  default  = []
  nullable = false

  description = "The log configurations of the service"
}

variable "service_tags" {
  type    = map(string)
  default = {}

  description = "The key/value tags to associate with the service"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts V2实时推理服务资源
resource "huaweicloud_modelartsv2_service" "test" {
  name         = var.service_name
  version      = var.service_version
  type         = var.service_type
  workspace_id = try(local.resource_pool.metadata[0].labels["os.modelarts/workspace.id"], null)
  deploy_type  = var.service_deploy_type
  description  = var.service_description

  group_configs {
    framework = var.service_group_framework
    name      = var.service_group_name
    pool_id   = try(local.resource_pool.metadata[0].name, null)
    weight    = var.service_group_weight
    count     = var.service_group_count

    dynamic "unit_configs" {
      for_each = var.service_unit_configs

      content {
        image {
          source   = unit_configs.value.image.source
          swr_path = unit_configs.value.image.swr_path
          id       = unit_configs.value.image.id
        }

        dynamic "custom_spec" {
          for_each = unit_configs.value.custom_spec != null ? [unit_configs.value.custom_spec] : []

          content {
            memory = custom_spec.value.memory
            cpu    = custom_spec.value.cpu
            gpu    = custom_spec.value.gpu
            ascend = custom_spec.value.ascend
          }
        }

        dynamic "models" {
          for_each = unit_configs.value.models

          content {
            source     = models.value.source
            mount_path = models.value.mount_path
            address    = models.value.address
            source_id  = models.value.source_id
          }
        }

        dynamic "codes" {
          for_each = unit_configs.value.codes

          content {
            source     = codes.value.source
            mount_path = codes.value.mount_path
            address    = codes.value.address
            source_id  = codes.value.source_id
          }
        }

        dynamic "readiness_health" {
          for_each = unit_configs.value.readiness_health != null ? [unit_configs.value.readiness_health] : []

          content {
            initial_delay_seconds = readiness_health.value.initial_delay_seconds
            timeout_seconds       = readiness_health.value.timeout_seconds
            period_seconds        = readiness_health.value.period_seconds
            failure_threshold     = readiness_health.value.failure_threshold
            check_method          = readiness_health.value.check_method
            command               = readiness_health.value.command
            url                   = readiness_health.value.url
          }
        }

        dynamic "startup_health" {
          for_each = unit_configs.value.startup_health != null ? [unit_configs.value.startup_health] : []

          content {
            initial_delay_seconds = startup_health.value.initial_delay_seconds
            timeout_seconds       = startup_health.value.timeout_seconds
            period_seconds        = startup_health.value.period_seconds
            failure_threshold     = startup_health.value.failure_threshold
            check_method          = startup_health.value.check_method
            command               = startup_health.value.command
            url                   = startup_health.value.url
          }
        }

        dynamic "liveness_health" {
          for_each = unit_configs.value.liveness_health != null ? [unit_configs.value.liveness_health] : []

          content {
            initial_delay_seconds = liveness_health.value.initial_delay_seconds
            timeout_seconds       = liveness_health.value.timeout_seconds
            period_seconds        = liveness_health.value.period_seconds
            failure_threshold     = liveness_health.value.failure_threshold
            check_method          = liveness_health.value.check_method
            command               = liveness_health.value.command
            url                   = liveness_health.value.url
          }
        }

        role     = unit_configs.value.role
        flavor   = unit_configs.value.flavor
        count    = unit_configs.value.count
        cmd      = unit_configs.value.cmd
        recovery = unit_configs.value.recovery
        envs     = unit_configs.value.envs
        port     = unit_configs.value.port
      }
    }
  }

  runtime_config = var.service_runtime_config
  upgrade_config = var.service_upgrade_config

  dynamic "log_configs" {
    for_each = var.service_log_configs

    content {
      type          = log_configs.value.type
      log_group_id  = log_configs.value.log_group_id
      log_stream_id = log_configs.value.log_stream_id
    }
  }

  tags = var.service_tags
}
```

**参数说明**：
- **name**：在线推理服务的名称，通过引用输入变量service_name进行赋值
- **version**：在线推理服务的版本，通过引用输入变量service_version进行赋值
- **type**：服务类型，通过引用输入变量service_type进行赋值，默认值为"REAL_TIME"
- **workspace_id**：工作空间ID，根据V2资源池列表查询数据源（data.huaweicloud_modelartsv2_resource_pools）返回结果中匹配资源池的metadata.labels进行赋值
- **deploy_type**：部署类型，通过引用输入变量service_deploy_type进行赋值，默认值为null
- **description**：在线推理服务的描述，通过引用输入变量service_description进行赋值，默认值为空字符串
- **group_configs.framework**：实例组的算法框架，通过引用输入变量service_group_framework进行赋值，默认值为"COMMON"
- **group_configs.name**：实例组的名称，通过引用输入变量service_group_name进行赋值
- **group_configs.pool_id**：实例组关联的专属资源池ID，根据V2资源池列表查询数据源返回结果中匹配资源池的metadata.name进行赋值
- **group_configs.weight**：实例组的权重百分比，通过引用输入变量service_group_weight进行赋值，默认值为100
- **group_configs.count**：部署场景下的服务实例数量，通过引用输入变量service_group_count进行赋值，默认值为1
- **group_configs.unit_configs.image.source**：单元镜像来源，通过service_unit_configs中对应项的image.source进行赋值
- **group_configs.unit_configs.image.swr_path**：单元SWR镜像路径，通过service_unit_configs中对应项的image.swr_path进行赋值
- **group_configs.unit_configs.image.id**：单元镜像ID，通过service_unit_configs中对应项的image.id进行赋值
- **group_configs.unit_configs.custom_spec.memory**：自定义规格内存大小，通过service_unit_configs中对应项的custom_spec.memory进行赋值
- **group_configs.unit_configs.custom_spec.cpu**：自定义规格CPU核数，通过service_unit_configs中对应项的custom_spec.cpu进行赋值
- **group_configs.unit_configs.custom_spec.gpu**：自定义规格GPU数量，通过service_unit_configs中对应项的custom_spec.gpu进行赋值
- **group_configs.unit_configs.custom_spec.ascend**：自定义规格昇腾卡数量，通过service_unit_configs中对应项的custom_spec.ascend进行赋值
- **group_configs.unit_configs.models.source**：模型来源，通过service_unit_configs中对应项的models.source进行赋值
- **group_configs.unit_configs.models.mount_path**：模型挂载路径，通过service_unit_configs中对应项的models.mount_path进行赋值
- **group_configs.unit_configs.models.address**：模型地址，通过service_unit_configs中对应项的models.address进行赋值
- **group_configs.unit_configs.models.source_id**：模型来源ID，通过service_unit_configs中对应项的models.source_id进行赋值
- **group_configs.unit_configs.codes.source**：代码来源，通过service_unit_configs中对应项的codes.source进行赋值
- **group_configs.unit_configs.codes.mount_path**：代码挂载路径，通过service_unit_configs中对应项的codes.mount_path进行赋值
- **group_configs.unit_configs.codes.address**：代码地址，通过service_unit_configs中对应项的codes.address进行赋值
- **group_configs.unit_configs.codes.source_id**：代码来源ID，通过service_unit_configs中对应项的codes.source_id进行赋值
- **group_configs.unit_configs.readiness_health.initial_delay_seconds**：就绪探针初始延迟时间（秒），通过service_unit_configs中对应项的readiness_health.initial_delay_seconds进行赋值
- **group_configs.unit_configs.readiness_health.timeout_seconds**：就绪探针超时时间（秒），通过service_unit_configs中对应项的readiness_health.timeout_seconds进行赋值
- **group_configs.unit_configs.readiness_health.period_seconds**：就绪探针检查周期（秒），通过service_unit_configs中对应项的readiness_health.period_seconds进行赋值
- **group_configs.unit_configs.readiness_health.failure_threshold**：就绪探针失败阈值，通过service_unit_configs中对应项的readiness_health.failure_threshold进行赋值
- **group_configs.unit_configs.readiness_health.check_method**：就绪探针检查方式，通过service_unit_configs中对应项的readiness_health.check_method进行赋值
- **group_configs.unit_configs.readiness_health.command**：就绪探针检查命令，通过service_unit_configs中对应项的readiness_health.command进行赋值
- **group_configs.unit_configs.readiness_health.url**：就绪探针检查URL，通过service_unit_configs中对应项的readiness_health.url进行赋值
- **group_configs.unit_configs.startup_health.initial_delay_seconds**：启动探针初始延迟时间（秒），通过service_unit_configs中对应项的startup_health.initial_delay_seconds进行赋值
- **group_configs.unit_configs.startup_health.timeout_seconds**：启动探针超时时间（秒），通过service_unit_configs中对应项的startup_health.timeout_seconds进行赋值
- **group_configs.unit_configs.startup_health.period_seconds**：启动探针检查周期（秒），通过service_unit_configs中对应项的startup_health.period_seconds进行赋值
- **group_configs.unit_configs.startup_health.failure_threshold**：启动探针失败阈值，通过service_unit_configs中对应项的startup_health.failure_threshold进行赋值
- **group_configs.unit_configs.startup_health.check_method**：启动探针检查方式，通过service_unit_configs中对应项的startup_health.check_method进行赋值
- **group_configs.unit_configs.startup_health.command**：启动探针检查命令，通过service_unit_configs中对应项的startup_health.command进行赋值
- **group_configs.unit_configs.startup_health.url**：启动探针检查URL，通过service_unit_configs中对应项的startup_health.url进行赋值
- **group_configs.unit_configs.liveness_health.initial_delay_seconds**：存活探针初始延迟时间（秒），通过service_unit_configs中对应项的liveness_health.initial_delay_seconds进行赋值
- **group_configs.unit_configs.liveness_health.timeout_seconds**：存活探针超时时间（秒），通过service_unit_configs中对应项的liveness_health.timeout_seconds进行赋值
- **group_configs.unit_configs.liveness_health.period_seconds**：存活探针检查周期（秒），通过service_unit_configs中对应项的liveness_health.period_seconds进行赋值
- **group_configs.unit_configs.liveness_health.failure_threshold**：存活探针失败阈值，通过service_unit_configs中对应项的liveness_health.failure_threshold进行赋值
- **group_configs.unit_configs.liveness_health.check_method**：存活探针检查方式，通过service_unit_configs中对应项的liveness_health.check_method进行赋值
- **group_configs.unit_configs.liveness_health.command**：存活探针检查命令，通过service_unit_configs中对应项的liveness_health.command进行赋值
- **group_configs.unit_configs.liveness_health.url**：存活探针检查URL，通过service_unit_configs中对应项的liveness_health.url进行赋值
- **group_configs.unit_configs.role**：单元角色，通过service_unit_configs中对应项的role进行赋值
- **group_configs.unit_configs.flavor**：单元规格，通过service_unit_configs中对应项的flavor进行赋值
- **group_configs.unit_configs.count**：单元实例数量，通过service_unit_configs中对应项的count进行赋值
- **group_configs.unit_configs.cmd**：单元启动命令，通过service_unit_configs中对应项的cmd进行赋值
- **group_configs.unit_configs.recovery**：单元故障恢复策略，通过service_unit_configs中对应项的recovery进行赋值
- **group_configs.unit_configs.envs**：单元环境变量，通过service_unit_configs中对应项的envs进行赋值
- **group_configs.unit_configs.port**：单元端口号，通过service_unit_configs中对应项的port进行赋值
- **runtime_config**：服务运行时配置，通过引用输入变量service_runtime_config进行赋值，JSON格式
- **upgrade_config**：服务升级配置，通过引用输入变量service_upgrade_config进行赋值，JSON格式
- **log_configs.type**：日志类型，通过service_log_configs中对应项的type进行赋值
- **log_configs.log_group_id**：LTS日志组ID，通过service_log_configs中对应项的log_group_id进行赋值
- **log_configs.log_stream_id**：LTS日志流ID，通过service_log_configs中对应项的log_stream_id进行赋值
- **tags**：服务标签，通过引用输入变量service_tags进行赋值，默认值为空映射

> service_group_pool_id需指定已存在的专属资源池ID。部署前请确保SWR镜像路径（swr_path）和单元规格（flavor）等参数与实际环境一致。

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`terraform.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 在线推理服务基本信息
service_name          = "tf_test_service"
service_description   = "Created online inference service by Terraform"
service_version       = "0.0.1"
service_deploy_type   = "MULTI"

# 实例组配置
service_group_name    = "tf_test_deploy_group"
service_group_pool_id = "pool-7b504f98-914a-4a03-93f1-d093f7b27908"

service_unit_configs = [
  {
    count    = 1
    recovery = "INSTANCE"
    role     = "COMMON"
    flavor   = "your_unit_resource_flavor" # 请替换为实际的单元资源规格

    image = {
      source   = "SWR"
      swr_path = "your_swr_image_path" # 请替换为实际的SWR镜像路径
    }

    readiness_health = {
      initial_delay_seconds = 60
      timeout_seconds       = 60
      period_seconds        = 30
      failure_threshold     = 6
      check_method          = "HTTP"
      url                   = "/health"
    }

    liveness_health = {
      initial_delay_seconds = 60
      timeout_seconds       = 60
      period_seconds        = 30
      failure_threshold     = 12
      check_method          = "HTTP"
      url                   = "/health"
    }

    startup_health = {
      initial_delay_seconds = 300
      timeout_seconds       = 60
      period_seconds        = 60
      failure_threshold     = 200
      check_method          = "HTTP"
      url                   = "/health"
    }
  }
]

service_runtime_config = <<-JSON
{
  "service_invoke": {
    "port": 8080,
    "protocol": "HTTPS",
    "auth_type": "TOKEN",
    "direct_channel_auth_enable": false
  },
  "service_limit": {
    "request_size_limit": 20,
    "request_timeout": 30,
    "ip_white_list": [],
    "ip_black_list": [],
    "rate_limit": {
      "num": 200,
      "unit": "SECONDS"
    }
  }
}
JSON

service_upgrade_config = <<-JSON
{
  "type": "ROLLING",
  "rolling_update": {
    "max_surge": "50%",
    "max_unavailable": "50%"
  }
}
JSON

service_log_configs = [
  {
    type = "STDOUT"
  }
]

service_tags = {
  source = "terraform"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="service_name=my-service" -var="service_version=1.0.0"`
2. 环境变量：`export TF_VAR_service_name=my-service`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建实时推理服务
4. 运行 `terraform show` 查看已创建的实时推理服务详情

## 参考信息

- [华为云ModelArts产品文档](https://support.huaweicloud.com/modelarts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ModelArts实时推理服务最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/modelarts/deploy-real-time-service)
