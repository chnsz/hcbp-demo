# 部署微调训练作业

## 应用场景

魔坊（ModelArts）是华为云提供的面向AI开发者的模型训推平台，支持大模型微调（Fine-Tuning）等多种AI训练能力。微调训练作业基于预训练模型和训练数据集，通过监督微调（SFT）等方式对模型进行定制化训练，适用于企业级大模型场景化落地。

本最佳实践适用于在ModelArts上使用公共资源池提交微调训练作业的场景，涵盖工作空间、SMN通知主题及微调训练作业的创建与配置，包括数据集输入、资产模型、输出模型、微调环境变量和检查点配置等。本最佳实践将介绍如何使用Terraform自动化部署上述资源，实现微调训练作业的Infrastructure as Code管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [ModelArts工作空间资源（huaweicloud_modelarts_workspace）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_workspace)
- [SMN主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [ModelArts训练作业资源（huaweicloud_modelarts_training_job）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_training_job)

### 资源/数据源依赖关系

```
huaweicloud_modelarts_workspace
    └── huaweicloud_modelarts_training_job

huaweicloud_smn_topic
    └── huaweicloud_modelarts_training_job
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建ModelArts工作空间（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts工作空间资源。当已通过workspace_id指定现有工作空间时可跳过此步骤：

```hcl
variable "workspace_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the workspace to create. Cannot be configured together with workspace_id."

  validation {
    condition     = var.workspace_name == "" || var.workspace_id == ""
    error_message = "workspace_name and workspace_id cannot be configured at the same time."
  }
}

variable "workspace_id" {
  type        = string
  default     = ""
  description = "The existing workspace ID. Cannot be configured together with workspace_name."
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts工作空间资源
resource "huaweicloud_modelarts_workspace" "test" {
  count = var.workspace_name != "" ? 1 : 0

  name = var.workspace_name
}
```

**参数说明**：
- **count**：资源的创建数，仅当workspace_name不为空时创建工作空间
- **name**：工作空间的名称，通过引用输入变量workspace_name进行赋值

> workspace_name与workspace_id不能同时配置。若使用已有工作空间，请配置workspace_id并留空workspace_name。

### 3. 创建SMN主题（可选）

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建SMN主题资源，用于训练作业通知。当已通过training_job_notification_topic_urn指定现有主题时可跳过此步骤：

```hcl
variable "topic_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the SMN topic to create for notifications. Cannot be configured together with training_job_notification_topic_urn."

  validation {
    condition     = var.topic_name == "" || var.training_job_notification_topic_urn == ""
    error_message = "topic_name and training_job_notification_topic_urn cannot be configured at the same time."
  }
}

variable "training_job_notification_topic_urn" {
  type        = string
  default     = ""
  nullable    = false
  description = "The existing SMN topic URN for training job notifications. Cannot be configured together with topic_name."
}

variable "training_job_notification_events" {
  type        = list(string)
  default     = []
  description = "The notification events of the training job"

  validation {
    condition     = (var.training_job_notification_topic_urn == "" && var.topic_name == "") || length(var.training_job_notification_events) > 0
    error_message = "training_job_notification_events is required when training_job_notification_topic_urn or topic_name is configured."
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SMN主题资源
resource "huaweicloud_smn_topic" "test" {
  count = var.topic_name != "" ? 1 : 0

  name = var.topic_name
}
```

**参数说明**：
- **count**：资源的创建数，仅当topic_name不为空时创建SMN主题
- **name**：SMN主题的名称，通过引用输入变量topic_name进行赋值

> topic_name与training_job_notification_topic_urn不能同时配置。配置topic_name或training_job_notification_topic_urn时，必须同时配置training_job_notification_events。

### 4. 创建ModelArts微调训练作业

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ModelArts微调训练作业资源：

```hcl
variable "training_job_train_type" {
  type        = string
  default     = "SFT"
  description = "The training type of the fine-tuning job"
}

variable "training_job_name" {
  type        = string
  description = "The name of the training job"
}

variable "training_job_annotations" {
  type        = map(string)
  default     = {}
  description = "The annotations of the training job"
}

variable "training_job_description" {
  type        = string
  default     = null
  description = "The description of the training job"
}

variable "training_job_inputs" {
  type = list(object({
    dataset = object({
      id                 = string
      name               = string
      version_id         = optional(string)
      service_type       = optional(string)
      dataset_proportion = optional(number)
    })

    local_dir = optional(string)
  }))

  default     = []
  nullable    = false
  description = "The inputs of the training job"
}

variable "training_job_environments" {
  type        = map(string)
  default     = null
  description = "The environment variables of the training job"
}

variable "resource_flavor_id" {
  type        = string
  default     = ""
  nullable    = false
  description = "The flavor ID of the public resource pool"
}

variable "resource_node_count" {
  type        = number
  default     = 1
  description = "The number of resource replicas used by the training job"
}

variable "training_job_asset_id" {
  type        = string
  description = "The asset ID of the training job"
}

variable "training_job_asset_model" {
  type = object({
    name    = string
    version = string
    type    = string
    code    = optional(string)
    desc    = optional(string)
    series  = optional(string)
  })

  description = "The asset model configuration of the training job"
}

variable "training_job_output_model" {
  type = object({
    obs_path   = string
    local_path = optional(string)
  })

  default     = null
  description = "The output model configuration of the training job"
}

variable "training_job_ftjob_config" {
  type = object({
    envs = list(object({
      env_name    = string
      env_type    = string
      value       = string
      label       = optional(string)
      des         = optional(string)
      modifiable  = optional(bool)
      displayable = optional(bool)
    }))

    checkpoint_config = optional(object({
      checkpoint_id        = optional(string)
      save_checkpoints_max = optional(number)
      skipped_steps        = optional(number)
      restore_training     = optional(number)
    }))
  })

  description = "The fine-tuning training job configuration"
}

variable "training_job_tags" {
  type        = map(string)
  default     = null
  description = "The tags of the training job"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ModelArts微调训练作业资源
resource "huaweicloud_modelarts_training_job" "test" {
  kind       = "job"
  train_type = var.training_job_train_type

  metadata {
    name         = var.training_job_name
    workspace_id = var.workspace_id != "" ? var.workspace_id : try(huaweicloud_modelarts_workspace.test[0].id, null)
    annotations  = var.training_job_annotations
    description  = var.training_job_description
  }

  algorithm {
    dynamic "inputs" {
      for_each = var.training_job_inputs

      content {
        local_dir = inputs.value.local_dir

        remote {
          dataset {
            id                 = inputs.value.dataset.id
            name               = inputs.value.dataset.name
            version_id         = inputs.value.dataset.version_id
            service_type       = inputs.value.dataset.service_type
            dataset_proportion = inputs.value.dataset.dataset_proportion
          }
        }
      }
    }

    environments = var.training_job_environments
  }

  spec {
    resource {
      flavor_id  = var.resource_flavor_id
      node_count = var.resource_node_count
    }

    asset_id = var.training_job_asset_id

    dynamic "asset_model" {
      for_each = [var.training_job_asset_model]

      content {
        name    = asset_model.value.name
        version = asset_model.value.version
        type    = asset_model.value.type
        code    = asset_model.value.code
        desc    = asset_model.value.desc
        series  = asset_model.value.series
      }
    }

    dynamic "notification" {
      for_each = var.training_job_notification_topic_urn != "" || var.topic_name != "" ? [1] : []

      content {
        topic_urn = var.training_job_notification_topic_urn != "" ? var.training_job_notification_topic_urn : var.topic_name != "" ? try(huaweicloud_smn_topic.test[0].id, null) : ""
        events    = var.training_job_notification_events
      }
    }

    dynamic "output_model" {
      for_each = var.training_job_output_model != null ? [var.training_job_output_model] : []

      content {
        obs {
          obs_path   = output_model.value.obs_path
          local_path = output_model.value.local_path
        }
      }
    }
  }

  ftjob_config {
    task_env {
      dynamic "envs" {
        for_each = var.training_job_ftjob_config.envs

        content {
          label       = envs.value.label
          des         = envs.value.des
          env_name    = envs.value.env_name
          env_type    = envs.value.env_type
          value       = envs.value.value
          modifiable  = envs.value.modifiable
          displayable = envs.value.displayable
        }
      }
    }

    dynamic "checkpoint_config" {
      for_each = var.training_job_ftjob_config.checkpoint_config != null ? [var.training_job_ftjob_config.checkpoint_config] : []

      content {
        checkpoint_id        = checkpoint_config.value.checkpoint_id
        save_checkpoints_max = checkpoint_config.value.save_checkpoints_max
        skipped_steps        = checkpoint_config.value.skipped_steps
        restore_training     = checkpoint_config.value.restore_training
      }
    }
  }

  tags = var.training_job_tags
}
```

**参数说明**：
- **kind**：训练作业类型，设置为"job"表示训练作业
- **train_type**：微调训练类型，通过引用输入变量training_job_train_type进行赋值，默认值为"SFT"
- **metadata.name**：训练作业的名称，通过引用输入变量training_job_name进行赋值
- **metadata.workspace_id**：工作空间ID，当workspace_id不为空时使用其值，否则引用创建的工作空间资源ID
- **metadata.annotations**：训练作业的注解，通过引用输入变量training_job_annotations进行赋值，默认值为空映射
- **metadata.description**：训练作业的描述，通过引用输入变量training_job_description进行赋值
- **algorithm.inputs**：训练输入配置，通过引用输入变量training_job_inputs进行赋值，支持数据集挂载及dataset_proportion比例配置
- **algorithm.environments**：环境变量，通过引用输入变量training_job_environments进行赋值
- **spec.resource.flavor_id**：公共资源池规格ID，通过引用输入变量resource_flavor_id进行赋值
- **spec.resource.node_count**：资源副本数，通过引用输入变量resource_node_count进行赋值，默认值为1
- **spec.asset_id**：训练资产ID，通过引用输入变量training_job_asset_id进行赋值
- **spec.asset_model**：资产模型配置，通过引用输入变量training_job_asset_model进行赋值，包含name、version、type等字段
- **spec.notification**：通知配置，当training_job_notification_topic_urn或topic_name不为空时配置SMN通知
- **spec.output_model**：输出模型配置，通过引用输入变量training_job_output_model进行赋值，支持OBS路径输出
- **ftjob_config.task_env.envs**：微调任务环境变量配置，通过引用输入变量training_job_ftjob_config.envs进行赋值
- **ftjob_config.checkpoint_config**：检查点配置，通过引用输入变量training_job_ftjob_config.checkpoint_config进行赋值，支持checkpoint_id、save_checkpoints_max、skipped_steps、restore_training等字段
- **tags**：训练作业标签，通过引用输入变量training_job_tags进行赋值

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`terraform.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 训练作业配置
training_job_name = "tf_test_training_job"

training_job_inputs = [
  {
    dataset = {
      id                 = "your-dataset-id"
      name               = "your-dataset-name"
      dataset_proportion = 100
    }
  }
]

resource_flavor_id    = "your-public-resource-pool-flavor"
training_job_asset_id = "your-asset-id"

training_job_asset_model = {
  name    = "new-fine-tuning"
  version = "1.0.0"
  type    = "NEW_ASSET"
  desc    = "assert new fine-tuning model"
}

training_job_output_model = {
  obs_path = "your-obs-path-for-output-model"
}

training_job_ftjob_config = {
  envs = [
    {
      label       = "MIN_LR"
      des         = "Minimum learning rate"
      env_name    = "MIN_LR"
      env_type    = "string"
      value       = "1.25e-7"
      modifiable  = true
      displayable = true
    },
    {
      label       = "LR"
      des         = "Learning rate"
      env_name    = "LR"
      env_type    = "string"
      value       = "1.25e-6"
      modifiable  = true
      displayable = true
    },
  ]

  checkpoint_config = {
    save_checkpoints_max = 5
  }
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="training_job_name=my-job" -var="training_job_asset_id=my-asset-id"`
2. 环境变量：`export TF_VAR_training_job_name=my-job`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建微调训练作业
4. 运行 `terraform show` 查看已创建的微调训练作业详情

## 参考信息

- [华为云ModelArts产品文档](https://support.huaweicloud.com/modelarts/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ModelArts微调训练作业最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/modelarts/fine-tuning-training-job)
