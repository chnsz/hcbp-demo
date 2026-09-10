# Deploy Flink Jar Job

## Application Scenario

Data Lake Insight (DLI) is a big data computing and analysis service provided by Huawei Cloud, supporting multiple computing engines such as SQL, Flink, and Spark, helping users quickly build data lake analysis businesses. This practice will guide you to use Terraform to create a DLI elastic resource pool, a general queue, and deploy a Flink Jar job on Huawei Cloud to implement stream data processing.

This best practice will introduce how to use Terraform to automatically deploy a DLI elastic resource pool, a general queue, and a Flink Jar job, including the CIDR configuration of the resource pool, the CU count setting of the queue, and the configuration of the JAR package path and Flink version of the job.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DLI Elastic Resource Pool (huaweicloud_dli_elastic_resource_pool)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_elastic_resource_pool)
- [DLI Queue (huaweicloud_dli_queue)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_queue)
- [DLI Flink Jar Job (huaweicloud_dli_flinkjar_job)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_flinkjar_job)

### Resource/Data Source Dependencies

```
huaweicloud_dli_elastic_resource_pool
    └── huaweicloud_dli_queue
        └── huaweicloud_dli_flinkjar_job
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the scripts of the current best practice in the specified working directory, and ensure that it (or other TF files in the same level directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create Elastic Resource Pool

Add the following script to the TF file (such as main.tf):

```hcl
# Create an elastic resource pool in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "elastic_resource_pool_name" {
  description = "The name of the elastic resource pool"
  type        = string
}

variable "elastic_resource_pool_cidr" {
  description = "The CIDR block of the elastic resource pool"
  type        = string
}

variable "elastic_resource_pool_description" {
  description = "The description of the elastic resource pool"
  type        = string
  default     = ""
}

variable "elastic_resource_pool_min_cu" {
  description = "The minimum number of CUs for the elastic resource pool"
  type        = number
  default     = 16
}

variable "elastic_resource_pool_max_cu" {
  description = "The maximum number of CUs for the elastic resource pool"
  type        = number
  default     = 64
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
}

variable "elastic_resource_pool_label" {
  description = "The label of the elastic resource pool"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_dli_elastic_resource_pool" "test" {
  name                  = var.elastic_resource_pool_name
  description           = var.elastic_resource_pool_description
  min_cu                = var.elastic_resource_pool_min_cu
  max_cu                = var.elastic_resource_pool_max_cu
  cidr                  = var.elastic_resource_pool_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  label                 = var.elastic_resource_pool_label
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable elastic_resource_pool_name, used to specify the name of the elastic resource pool.
- **description**: Assigned by referencing the input variable elastic_resource_pool_description, used to describe the elastic resource pool.
- **min_cu**: Assigned by referencing the input variable elastic_resource_pool_min_cu, used to specify the minimum number of CUs for the elastic resource pool.
- **max_cu**: Assigned by referencing the input variable elastic_resource_pool_max_cu, used to specify the maximum number of CUs for the elastic resource pool.
- **cidr**: Assigned by referencing the input variable elastic_resource_pool_cidr, used to specify the CIDR block of the elastic resource pool.
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, and set when the variable value is not empty.
- **label**: Assigned by referencing the input variable elastic_resource_pool_label, used to add tags to the elastic resource pool.

### 3. Create General Queue

Add the following script to the TF file (such as main.tf):

```hcl
# Create a general queue in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "queue_name" {
  description = "The name of the DLI general queue"
  type        = string
}

variable "queue_cu_count" {
  description = "The CU count of the DLI queue"
  type        = number
  default     = 16
}

variable "queue_description" {
  description = "The description of the DLI queue"
  type        = string
  default     = ""
}

resource "huaweicloud_dli_queue" "test" {
  elastic_resource_pool_name = huaweicloud_dli_elastic_resource_pool.test.name
  resource_mode              = 1
  name                       = var.queue_name
  queue_type                 = "general"
  cu_count                   = var.queue_cu_count
  description                = var.queue_description
  enterprise_project_id      = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:
- **elastic_resource_pool_name**: Assigned by referencing the name of the created elastic resource pool, associating the queue with that resource pool.
- **resource_mode**: Fixed to 1, indicating that the queue type is dedicated resource pool mode.
- **name**: Assigned by referencing the input variable queue_name, used to specify the name of the queue.
- **queue_type**: Fixed to "general", indicating that the queue type is a general queue.
- **cu_count**: Assigned by referencing the input variable queue_cu_count, used to specify the CU count of the queue.
- **description**: Assigned by referencing the input variable queue_description, used to describe the queue.
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, and set when the variable value is not empty.

### 4. Create Flink Jar Job

Add the following script to the TF file (such as main.tf):

```hcl
# Create a Flink Jar job in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "job_name" {
  description = "The name of the Flink Jar job"
  type        = string
}

variable "job_description" {
  description = "The description of the Flink Jar job"
  type        = string
  default     = ""
}

variable "job_main_class" {
  description = "The main class of the Flink Jar job"
  type        = string
  default     = ""
}

variable "job_entrypoint" {
  description = "The OBS path of the JAR file"
  type        = string
}

variable "job_entrypoint_args" {
  description = "The entrypoint arguments of the Flink Jar job"
  type        = string
  default     = ""
}

variable "job_dependency_jars" {
  description = "The dependency JAR packages of the Flink Jar job"
  type        = list(string)
  default     = []
}

variable "job_dependency_files" {
  description = "The dependency files of the Flink Jar job"
  type        = list(string)
  default     = []
}

variable "job_execution_agency_urn" {
  description = "The agency URN authorized to DLI"
  type        = string
  default     = ""
}

variable "job_feature" {
  description = "The feature type of the Flink image. Valid values are basic and custom"
  type        = string
  default     = "basic"
}

variable "job_flink_version" {
  description = "The Flink version"
  type        = string
}

variable "job_image" {
  description = "The custom Flink image. Available when feature is custom"
  type        = string
  default     = ""
}

variable "job_cu_num" {
  description = "The number of CUs selected for the Flink Jar job"
  type        = number
  default     = 2
}

variable "job_parallel_num" {
  description = "The parallelism of the Flink Jar job"
  type        = number
  default     = 1
}

variable "job_obs_bucket" {
  description = "The OBS bucket used to save logs"
  type        = string
  default     = ""
}

variable "job_log_enabled" {
  description = "Whether to enable uploading job logs to the OBS bucket"
  type        = bool
  default     = false
}

variable "job_smn_topic" {
  description = "The SMN topic used to receive job failure notifications"
  type        = string
  default     = ""
}

variable "job_restart_when_exception" {
  description = "Whether to enable the function of automatically restarting a job upon job exceptions"
  type        = bool
  default     = false
}

variable "job_manager_cu_num" {
  description = "The number of CUs in the JobManager"
  type        = number
  default     = 1
}

variable "job_tm_cu_num" {
  description = "The number of CUs for each Task Manager"
  type        = number
  default     = 1
}

variable "job_tm_slot_num" {
  description = "The number of slots in each Task Manager"
  type        = number
  default     = null
}

variable "job_resume_checkpoint" {
  description = "Whether to resume from the checkpoint upon exceptions"
  type        = bool
  default     = false
}

variable "job_resume_max_num" {
  description = "The maximum number of retry times upon exceptions"
  type        = number
  default     = -1
}

variable "job_runtime_config" {
  description = "The custom optimization parameters when the Flink job is running"
  type        = map(string)
  default     = {}
}

variable "job_checkpoint_path" {
  description = "The OBS path used to store checkpoints"
  type        = string
  default     = ""
}

variable "job_tags" {
  description = "The key/value pairs to associate with the Flink Jar job"
  type        = map(string)
  default     = {}
}

variable "job_checkpoint_enabled" {
  description = "Whether to enable the automatic job snapshot function"
  type        = bool
  default     = false
}

variable "job_checkpoint_mode" {
  description = "The snapshot mode"
  type        = number
  default     = 1
}

variable "job_checkpoint_interval" {
  description = "The snapshot interval in seconds"
  type        = number
  default     = 30
}

resource "huaweicloud_dli_flinkjar_job" "test" {
  name                   = var.job_name
  description            = var.job_description
  queue_name             = huaweicloud_dli_queue.test.name
  main_class             = var.job_main_class
  entrypoint             = var.job_entrypoint
  entrypoint_args        = var.job_entrypoint_args
  dependency_jars        = var.job_dependency_jars
  dependency_files       = var.job_dependency_files
  execution_agency_urn   = var.job_execution_agency_urn
  feature                = var.job_feature
  flink_version          = var.job_flink_version
  image                  = var.job_image
  cu_num                 = var.job_cu_num
  parallel_num           = var.job_parallel_num
  obs_bucket             = var.job_obs_bucket
  log_enabled            = var.job_log_enabled
  smn_topic              = var.job_smn_topic
  restart_when_exception = var.job_restart_when_exception
  manager_cu_num         = var.job_manager_cu_num
  tm_cu_num              = var.job_tm_cu_num
  tm_slot_num            = var.job_tm_slot_num
  resume_checkpoint      = var.job_resume_checkpoint
  resume_max_num         = var.job_resume_max_num
  runtime_config         = var.job_runtime_config
  checkpoint_path        = var.job_checkpoint_path
  tags                   = var.job_tags
  checkpoint_enabled     = var.job_checkpoint_enabled
  checkpoint_mode        = var.job_checkpoint_mode
  checkpoint_interval    = var.job_checkpoint_interval

  depends_on = [huaweicloud_dli_queue.test]
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable job_name, used to specify the name of the Flink Jar job.
- **description**: Assigned by referencing the input variable job_description, used to describe the job.
- **queue_name**: Assigned by referencing the name of the created DLI queue, specifying the queue on which the job runs.
- **main_class**: Assigned by referencing the input variable job_main_class, specifying the main class of the job.
- **entrypoint**: Assigned by referencing the input variable job_entrypoint, specifying the OBS path of the JAR file.
- **entrypoint_args**: Assigned by referencing the input variable job_entrypoint_args, specifying the entrypoint arguments of the job.
- **dependency_jars**: Assigned by referencing the input variable job_dependency_jars, specifying the list of dependency JAR packages.
- **dependency_files**: Assigned by referencing the input variable job_dependency_files, specifying the list of dependency files.
- **execution_agency_urn**: Assigned by referencing the input variable job_execution_agency_urn, specifying the agency URN authorized to DLI.
- **feature**: Assigned by referencing the input variable job_feature, specifying the feature type of the Flink image.
- **flink_version**: Assigned by referencing the input variable job_flink_version, specifying the Flink version.
- **image**: Assigned by referencing the input variable job_image, specifying the custom Flink image.
- **cu_num**: Assigned by referencing the input variable job_cu_num, specifying the number of CUs for the job.
- **parallel_num**: Assigned by referencing the input variable job_parallel_num, specifying the parallelism of the job.
- **obs_bucket**: Assigned by referencing the input variable job_obs_bucket, specifying the OBS bucket used to save logs.
- **log_enabled**: Assigned by referencing the input variable job_log_enabled, specifying whether to enable log upload.
- **smn_topic**: Assigned by referencing the input variable job_smn_topic, specifying the SMN topic for failure notifications.
- **restart_when_exception**: Assigned by referencing the input variable job_restart_when_exception, specifying whether to automatically restart on exceptions.
- **manager_cu_num**: Assigned by referencing the input variable job_manager_cu_num, specifying the number of CUs in the JobManager.
- **tm_cu_num**: Assigned by referencing the input variable job_tm_cu_num, specifying the number of CUs for each Task Manager.
- **tm_slot_num**: Assigned by referencing the input variable job_tm_slot_num, specifying the number of slots in each Task Manager.
- **resume_checkpoint**: Assigned by referencing the input variable job_resume_checkpoint, specifying whether to resume from the checkpoint on exceptions.
- **resume_max_num**: Assigned by referencing the input variable job_resume_max_num, specifying the maximum number of retry times on exceptions.
- **runtime_config**: Assigned by referencing the input variable job_runtime_config, specifying custom optimization parameters at runtime.
- **checkpoint_path**: Assigned by referencing the input variable job_checkpoint_path, specifying the OBS path for checkpoints.
- **tags**: Assigned by referencing the input variable job_tags, used to add tags to the job.
- **checkpoint_enabled**: Assigned by referencing the input variable job_checkpoint_enabled, specifying whether to enable automatic snapshots.
- **checkpoint_mode**: Assigned by referencing the input variable job_checkpoint_mode, specifying the snapshot mode.
- **checkpoint_interval**: Assigned by referencing the input variable job_checkpoint_interval, specifying the snapshot interval.

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to the configuration. These input parameters need to be manually entered during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, avoiding repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, for example:

```hcl
# Fill in according to the script variables; use placeholders for sensitive information
elastic_resource_pool_name = "your_elastic_resource_pool_name"
elastic_resource_pool_cidr = "172.16.0.0/18"
queue_name                 = "your_general_queue_name"
job_name                   = "your_flink_jar_job_name"
job_entrypoint             = "obs://your_bucket_path/your.jar"
job_flink_version          = "1.15"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing terraform commands; other names need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the Flink Jar job
4. Run `terraform show` to view the created Flink Jar job

## Reference Information

- [Huawei Cloud Data Lake Insight Product Documentation](https://support.huaweicloud.com/dli/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DLI Flink Jar Job](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dli/flink-jar-job)
