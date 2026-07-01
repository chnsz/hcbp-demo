# Deploy Real-Time Inference Service

## Application Scenario

The ModelArts online inference service supports deploying trained models as real-time inference services, providing low-latency and highly available model inference capabilities. Real-time inference services (REAL_TIME) are suitable for business scenarios sensitive to response latency, supporting enterprise features such as multi-instance group deployment, health checks, rolling upgrades, and log collection.

This best practice applies to deploying a real-time inference service on a ModelArts dedicated resource pool, covering V2 resource pool query, online inference service creation, instance group and unit configuration, runtime configuration, upgrade strategy, and log configuration. This best practice introduces how to use Terraform to automatically deploy the above resources and manage real-time inference services with Infrastructure as Code.

## Related Resources and Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [ModelArts V2 Resource Pools Data Source (data.huaweicloud_modelartsv2_resource_pools)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelartsv2_resource_pools)

### Resources

- [ModelArts V2 Online Inference Service Resource (huaweicloud_modelartsv2_service)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelartsv2_service)

### Resource/Data Source Dependency Relationships

```
data.huaweicloud_modelartsv2_resource_pools
    └── huaweicloud_modelartsv2_service
```

## Operation Steps

### 1. Script Preparation

Prepare TF files (such as main.tf) in the specified workspace for writing the current best practice scripts. Ensure that they (or other TF files in the same directory) contain the provider version declaration and Huawei Cloud authentication information required for resource deployment.
For configuration details, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query ModelArts V2 Resource Pool Information

Add the following script to the TF file (such as main.tf) to instruct Terraform to query ModelArts V2 resource pool information. The query results are used to obtain the workspace ID and instance group pool_id corresponding to the dedicated resource pool:

```hcl
variable "service_group_pool_id" {
  type        = string
  description = "The ID of the dedicated resource pool for the instance group"
}

# Query ModelArts V2 resource pool information in the specified region (inherits the region from the current provider block when the region parameter is omitted), used to obtain workspace ID and instance group pool_id
data "huaweicloud_modelartsv2_resource_pools" "test" {}

locals {
  resource_pool = try([for pool in data.huaweicloud_modelartsv2_resource_pools.test.resource_pools : pool if pool.metadata[0].name ==
  var.service_group_pool_id][0], {})
}
```

**Parameter Description**:
- This data source requires no additional parameters and automatically queries all V2 resource pools in the current region
- **service_group_pool_id**: Dedicated resource pool ID used by the instance group, assigned by referencing the input variable service_group_pool_id
- **resource_pool**: Local variable that matches the corresponding V2 resource pool information based on service_group_pool_id

### 3. Create ModelArts V2 Real-Time Inference Service

Add the following script to the TF file (such as main.tf) to instruct Terraform to create a ModelArts V2 real-time inference service resource:

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

# Create a ModelArts V2 real-time inference service resource in the specified region (inherits the region from the current provider block when the region parameter is omitted)
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

**Parameter Description**:
- **name**: Online inference service name, assigned by referencing the input variable service_name
- **version**: Online inference service version, assigned by referencing the input variable service_version
- **type**: Service type, assigned by referencing the input variable service_type, default value is "REAL_TIME"
- **workspace_id**: Workspace ID, assigned based on the metadata.labels of the matched resource pool in the return result of the V2 resource pools data source (data.huaweicloud_modelartsv2_resource_pools)
- **deploy_type**: Deployment type, assigned by referencing the input variable service_deploy_type, default value is null
- **description**: Online inference service description, assigned by referencing the input variable service_description, default value is an empty string
- **group_configs.framework**: Algorithm framework of the instance group, assigned by referencing the input variable service_group_framework, default value is "COMMON"
- **group_configs.name**: Instance group name, assigned by referencing the input variable service_group_name
- **group_configs.pool_id**: Dedicated resource pool ID associated with the instance group, assigned based on the metadata.name of the matched resource pool in the V2 resource pools data source return result
- **group_configs.weight**: Weight percentage of the instance group, assigned by referencing the input variable service_group_weight, default value is 100
- **group_configs.count**: Number of service instances in the deployment scenario, assigned by referencing the input variable service_group_count, default value is 1
- **group_configs.unit_configs.image.source**: Unit image source, assigned from the image.source in the corresponding item in service_unit_configs
- **group_configs.unit_configs.image.swr_path**: Unit SWR image path, assigned from the image.swr_path in the corresponding item in service_unit_configs
- **group_configs.unit_configs.image.id**: Unit image ID, assigned from the image.id in the corresponding item in service_unit_configs
- **group_configs.unit_configs.custom_spec.memory**: Custom specification memory size, assigned from the custom_spec.memory in the corresponding item in service_unit_configs
- **group_configs.unit_configs.custom_spec.cpu**: Custom specification CPU cores, assigned from the custom_spec.cpu in the corresponding item in service_unit_configs
- **group_configs.unit_configs.custom_spec.gpu**: Custom specification GPU count, assigned from the custom_spec.gpu in the corresponding item in service_unit_configs
- **group_configs.unit_configs.custom_spec.ascend**: Custom specification Ascend card count, assigned from the custom_spec.ascend in the corresponding item in service_unit_configs
- **group_configs.unit_configs.models.source**: Model source, assigned from the models.source in the corresponding item in service_unit_configs
- **group_configs.unit_configs.models.mount_path**: Model mount path, assigned from the models.mount_path in the corresponding item in service_unit_configs
- **group_configs.unit_configs.models.address**: Model address, assigned from the models.address in the corresponding item in service_unit_configs
- **group_configs.unit_configs.models.source_id**: Model source ID, assigned from the models.source_id in the corresponding item in service_unit_configs
- **group_configs.unit_configs.codes.source**: Code source, assigned from the codes.source in the corresponding item in service_unit_configs
- **group_configs.unit_configs.codes.mount_path**: Code mount path, assigned from the codes.mount_path in the corresponding item in service_unit_configs
- **group_configs.unit_configs.codes.address**: Code address, assigned from the codes.address in the corresponding item in service_unit_configs
- **group_configs.unit_configs.codes.source_id**: Code source ID, assigned from the codes.source_id in the corresponding item in service_unit_configs
- **group_configs.unit_configs.readiness_health.initial_delay_seconds**: Readiness probe initial delay (seconds), assigned from the readiness_health.initial_delay_seconds in the corresponding item in service_unit_configs
- **group_configs.unit_configs.readiness_health.timeout_seconds**: Readiness probe timeout (seconds), assigned from the readiness_health.timeout_seconds in the corresponding item in service_unit_configs
- **group_configs.unit_configs.readiness_health.period_seconds**: Readiness probe period (seconds), assigned from the readiness_health.period_seconds in the corresponding item in service_unit_configs
- **group_configs.unit_configs.readiness_health.failure_threshold**: Readiness probe failure threshold, assigned from the readiness_health.failure_threshold in the corresponding item in service_unit_configs
- **group_configs.unit_configs.readiness_health.check_method**: Readiness probe check method, assigned from the readiness_health.check_method in the corresponding item in service_unit_configs
- **group_configs.unit_configs.readiness_health.command**: Readiness probe check command, assigned from the readiness_health.command in the corresponding item in service_unit_configs
- **group_configs.unit_configs.readiness_health.url**: Readiness probe check URL, assigned from the readiness_health.url in the corresponding item in service_unit_configs
- **group_configs.unit_configs.startup_health.initial_delay_seconds**: Startup probe initial delay (seconds), assigned from the startup_health.initial_delay_seconds in the corresponding item in service_unit_configs
- **group_configs.unit_configs.startup_health.timeout_seconds**: Startup probe timeout (seconds), assigned from the startup_health.timeout_seconds in the corresponding item in service_unit_configs
- **group_configs.unit_configs.startup_health.period_seconds**: Startup probe period (seconds), assigned from the startup_health.period_seconds in the corresponding item in service_unit_configs
- **group_configs.unit_configs.startup_health.failure_threshold**: Startup probe failure threshold, assigned from the startup_health.failure_threshold in the corresponding item in service_unit_configs
- **group_configs.unit_configs.startup_health.check_method**: Startup probe check method, assigned from the startup_health.check_method in the corresponding item in service_unit_configs
- **group_configs.unit_configs.startup_health.command**: Startup probe check command, assigned from the startup_health.command in the corresponding item in service_unit_configs
- **group_configs.unit_configs.startup_health.url**: Startup probe check URL, assigned from the startup_health.url in the corresponding item in service_unit_configs
- **group_configs.unit_configs.liveness_health.initial_delay_seconds**: Liveness probe initial delay (seconds), assigned from the liveness_health.initial_delay_seconds in the corresponding item in service_unit_configs
- **group_configs.unit_configs.liveness_health.timeout_seconds**: Liveness probe timeout (seconds), assigned from the liveness_health.timeout_seconds in the corresponding item in service_unit_configs
- **group_configs.unit_configs.liveness_health.period_seconds**: Liveness probe period (seconds), assigned from the liveness_health.period_seconds in the corresponding item in service_unit_configs
- **group_configs.unit_configs.liveness_health.failure_threshold**: Liveness probe failure threshold, assigned from the liveness_health.failure_threshold in the corresponding item in service_unit_configs
- **group_configs.unit_configs.liveness_health.check_method**: Liveness probe check method, assigned from the liveness_health.check_method in the corresponding item in service_unit_configs
- **group_configs.unit_configs.liveness_health.command**: Liveness probe check command, assigned from the liveness_health.command in the corresponding item in service_unit_configs
- **group_configs.unit_configs.liveness_health.url**: Liveness probe check URL, assigned from the liveness_health.url in the corresponding item in service_unit_configs
- **group_configs.unit_configs.role**: Unit role, assigned from the role in the corresponding item in service_unit_configs
- **group_configs.unit_configs.flavor**: Unit flavor, assigned from the flavor in the corresponding item in service_unit_configs
- **group_configs.unit_configs.count**: Unit instance count, assigned from the count in the corresponding item in service_unit_configs
- **group_configs.unit_configs.cmd**: Unit startup command, assigned from the cmd in the corresponding item in service_unit_configs
- **group_configs.unit_configs.recovery**: Unit fault recovery strategy, assigned from the recovery in the corresponding item in service_unit_configs
- **group_configs.unit_configs.envs**: Unit environment variables, assigned from the envs in the corresponding item in service_unit_configs
- **group_configs.unit_configs.port**: Unit port number, assigned from the port in the corresponding item in service_unit_configs
- **runtime_config**: Service runtime configuration, assigned by referencing the input variable service_runtime_config, in JSON format
- **upgrade_config**: Service upgrade configuration, assigned by referencing the input variable service_upgrade_config, in JSON format
- **log_configs.type**: Log type, assigned from the type in the corresponding item in service_log_configs
- **log_configs.log_group_id**: LTS log group ID, assigned from the log_group_id in the corresponding item in service_log_configs
- **log_configs.log_stream_id**: LTS log stream ID, assigned from the log_stream_id in the corresponding item in service_log_configs
- **tags**: Service tags, assigned by referencing the input variable service_tags, default value is an empty map

> service_group_pool_id must specify an existing dedicated resource pool ID. Before deployment, ensure parameters such as SWR image path (swr_path) and unit flavor (flavor) match the actual environment.

### 4. Preset Input Parameters for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through the `terraform.tfvars` file to avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Online inference service basic information
service_name          = "tf_test_service"
service_description   = "Created online inference service by Terraform"
service_version       = "0.0.1"
service_deploy_type   = "MULTI"

# Instance group configuration
service_group_name    = "tf_test_deploy_group"
service_group_pool_id = "pool-7b504f98-914a-4a03-93f1-d093f7b27908"

service_unit_configs = [
  {
    count    = 1
    recovery = "INSTANCE"
    role     = "COMMON"
    flavor   = "your_unit_resource_flavor" # Please replace with your actual unit resource flavor

    image = {
      source   = "SWR"
      swr_path = "your_swr_image_path" # Please replace with your actual SWR image path
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

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the contents of this `tfvars` file when executing terraform commands; other names require adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read variable values from this file

In addition to using the `terraform.tfvars` file, variable values can also be set through the following methods:

1. Command line parameters: `terraform apply -var="service_name=my-service" -var="service_version=1.0.0"`
2. Environment variables: `export TF_VAR_service_name=my-service`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform uses variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the real-time inference service
4. Run `terraform show` to view the details of the created real-time inference service

## Reference Information

- [Huawei Cloud ModelArts Product Documentation](https://support.huaweicloud.com/modelarts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ModelArts Real-Time Inference Service](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/modelarts/deploy-real-time-service)
