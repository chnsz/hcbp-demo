# Deploy Fine-Tuning Training Job

## Application Scenario

ModelArts is a model training and inference platform provided by Huawei Cloud for AI developers, supporting AI training capabilities such as large model fine-tuning. Fine-tuning training jobs customize models based on pre-trained models and training datasets through methods such as Supervised Fine-Tuning (SFT), suitable for enterprise large model scenario deployment.

This best practice is suitable for scenarios where you need to submit a fine-tuning training job using a public resource pool on ModelArts, covering the creation and configuration of workspace, SMN notification topic, and fine-tuning training job, including dataset inputs, asset model, output model, fine-tuning environment variables, and checkpoint configuration. This best practice will introduce how to use Terraform to automatically deploy the above resources for Infrastructure as Code management of fine-tuning training jobs.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [ModelArts Workspace Resource (huaweicloud_modelarts_workspace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_workspace)
- [SMN Topic Resource (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [ModelArts Training Job Resource (huaweicloud_modelarts_training_job)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_training_job)

### Resource/Data Source Dependencies

```
huaweicloud_modelarts_workspace
    └── huaweicloud_modelarts_training_job

huaweicloud_smn_topic
    └── huaweicloud_modelarts_training_job
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Create ModelArts Workspace (Optional)

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a ModelArts workspace resource. Skip this step when an existing workspace is specified via workspace_id:

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

# Create a ModelArts workspace resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_modelarts_workspace" "test" {
  count = var.workspace_name != "" ? 1 : 0

  name = var.workspace_name
}
```

**Parameter Description**:
- **count**: Number of resources to create, creates a workspace only when workspace_name is not empty
- **name**: Workspace name, assigned by referencing the input variable workspace_name

> workspace_name and workspace_id cannot be configured at the same time. To use an existing workspace, configure workspace_id and leave workspace_name empty.

### 3. Create SMN Topic (Optional)

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an SMN topic resource for training job notifications. Skip this step when an existing topic is specified via training_job_notification_topic_urn:

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

# Create an SMN topic resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_smn_topic" "test" {
  count = var.topic_name != "" ? 1 : 0

  name = var.topic_name
}
```

**Parameter Description**:
- **count**: Number of resources to create, creates an SMN topic only when topic_name is not empty
- **name**: SMN topic name, assigned by referencing the input variable topic_name

> topic_name and training_job_notification_topic_urn cannot be configured at the same time. When topic_name or training_job_notification_topic_urn is configured, training_job_notification_events must also be configured.

### 4. Create ModelArts Fine-Tuning Training Job

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a ModelArts fine-tuning training job resource:

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

# Create a ModelArts fine-tuning training job resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
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

**Parameter Description**:
- **kind**: Training job type, set to "job" for training jobs
- **train_type**: Fine-tuning training type, assigned by referencing the input variable training_job_train_type, default value is "SFT"
- **metadata.name**: Training job name, assigned by referencing the input variable training_job_name
- **metadata.workspace_id**: Workspace ID, uses workspace_id when it is not empty, otherwise references the created workspace resource ID
- **metadata.annotations**: Training job annotations, assigned by referencing the input variable training_job_annotations, default value is an empty map
- **metadata.description**: Training job description, assigned by referencing the input variable training_job_description
- **algorithm.inputs**: Training input configuration, assigned by referencing the input variable training_job_inputs, supporting dataset mounting and dataset_proportion configuration
- **algorithm.environments**: Environment variables, assigned by referencing the input variable training_job_environments
- **spec.resource.flavor_id**: Public resource pool flavor ID, assigned by referencing the input variable resource_flavor_id
- **spec.resource.node_count**: Number of resource replicas, assigned by referencing the input variable resource_node_count, default value is 1
- **spec.asset_id**: Training asset ID, assigned by referencing the input variable training_job_asset_id
- **spec.asset_model**: Asset model configuration, assigned by referencing the input variable training_job_asset_model, including fields such as name, version, and type
- **spec.notification**: Notification configuration, configured when training_job_notification_topic_urn or topic_name is not empty
- **spec.output_model**: Output model configuration, assigned by referencing the input variable training_job_output_model, supporting OBS path output
- **ftjob_config.task_env.envs**: Fine-tuning task environment variable configuration, assigned by referencing the input variable training_job_ftjob_config.envs
- **ftjob_config.checkpoint_config**: Checkpoint configuration, assigned by referencing the input variable training_job_ftjob_config.checkpoint_config, supporting fields such as checkpoint_id, save_checkpoints_max, skipped_steps, and restore_training
- **tags**: Training job tags, assigned by referencing the input variable training_job_tags

### 5. Preset Input Parameters for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration values. These input parameters need to be entered manually during subsequent deployment.
Terraform also provides a method to preset these configurations through a `terraform.tfvars` file to avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Training job configuration
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

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the contents when executing terraform commands; other names require adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read variable values from this file

In addition to using a `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="training_job_name=my-job" -var="training_job_asset_id=my-asset-id"`
2. Environment variables: `export TF_VAR_training_job_name=my-job`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform uses variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the fine-tuning training job
4. Run `terraform show` to view details of the created fine-tuning training job

## Reference Information

- [Huawei Cloud ModelArts Product Documentation](https://support.huaweicloud.com/modelarts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ModelArts Fine-Tuning Training Job](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/modelarts/fine-tuning-training-job)
