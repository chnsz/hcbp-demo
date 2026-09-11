# Deploy Flink OpenSource SQL Job

## Application Scenario

Data Lake Insight (DLI) is a big data computing and analysis service provided by Huawei Cloud, supporting multiple computing engines such as SQL, Flink, and Spark. A Flink OpenSource SQL job allows users to describe data sources, query logic, and result outputs using standard Flink SQL statements, enabling stream data processing without writing Java/Scala code.

This best practice will introduce how to use Terraform to automatically deploy a DLI elastic resource pool, a general queue, and a Flink OpenSource SQL job, including elastic resource pool creation, queue configuration, and job parameter settings.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DLI Elastic Resource Pool (huaweicloud_dli_elastic_resource_pool)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_elastic_resource_pool)
- [DLI Queue (huaweicloud_dli_queue)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_queue)
- [DLI Flink SQL Job (huaweicloud_dli_flinksql_job)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_flinksql_job)

### Resource/Data Source Dependencies

```
huaweicloud_dli_elastic_resource_pool
    └── huaweicloud_dli_queue
            └── huaweicloud_dli_flinksql_job
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a DLI Elastic Resource Pool

Add the following script to the TF file (such as main.tf) to create a DLI elastic resource pool:

```hcl
# Create a DLI elastic resource pool in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "elastic_resource_pool_name" {
  description = "The name of the DLI elastic resource pool"
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

variable "elastic_resource_pool_cidr" {
  description = "The CIDR block of the elastic resource pool."
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
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
- **name**: The name of the elastic resource pool, assigned by referencing the input variable elastic_resource_pool_name
- **description**: The description of the elastic resource pool, assigned by referencing the input variable elastic_resource_pool_description
- **min_cu**: The minimum number of CUs for the elastic resource pool, assigned by referencing the input variable elastic_resource_pool_min_cu
- **max_cu**: The maximum number of CUs for the elastic resource pool, assigned by referencing the input variable elastic_resource_pool_max_cu
- **cidr**: The CIDR block of the elastic resource pool, assigned by referencing the input variable elastic_resource_pool_cidr
- **enterprise_project_id**: The ID of the enterprise project, assigned by referencing the input variable enterprise_project_id, using null when empty
- **label**: The label of the elastic resource pool, assigned by referencing the input variable elastic_resource_pool_label

### 3. Create a DLI Queue

Add the following script to the TF file to create a DLI general queue:

```hcl
# Create a DLI queue in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "queue_name" {
  description = "The name of the DLI general queue used to run the Flink OpenSource SQL job"
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
- **elastic_resource_pool_name**: The name of the elastic resource pool to which the queue belongs, assigned by referencing the name of the elastic resource pool created in the previous step
- **resource_mode**: The resource mode, fixed to 1 (elastic resource pool mode)
- **name**: The name of the queue, assigned by referencing the input variable queue_name
- **queue_type**: The type of the queue, fixed to general
- **cu_count**: The CU count of the queue, assigned by referencing the input variable queue_cu_count
- **description**: The description of the queue, assigned by referencing the input variable queue_description
- **enterprise_project_id**: The ID of the enterprise project, assigned by referencing the input variable enterprise_project_id, using null when empty

### 4. Create a Flink OpenSource SQL Job

Add the following script to the TF file to create a Flink OpenSource SQL job:

```hcl
# Create a Flink OpenSource SQL job in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "job_name" {
  description = "The name of the Flink OpenSource SQL job"
  type        = string
}

variable "job_sql" {
  description = "The Flink SQL statement that includes source, query and sink"
  type        = string
}

variable "job_flink_version" {
  description = "The Flink version"
  type        = string
}

variable "job_execution_agency_urn" {
  description = "The agency URN authorized to DLI"
  type        = string
  default     = ""
}

variable "job_cu_number" {
  description = "The number of CUs selected for the Flink OpenSource SQL job"
  type        = number
  default     = 2
}

variable "job_parallel_number" {
  description = "The parallelism of the Flink OpenSource SQL job"
  type        = number
  default     = 1
}

variable "job_manager_cu_number" {
  description = "The number of CUs in the JobManager"
  type        = number
  default     = 1
}

variable "job_tm_cus" {
  description = "The number of CUs for each Task Manager"
  type        = number
  default     = 1
}

variable "job_tm_slot_num" {
  description = "The number of slots in each Task Manager"
  type        = number
  default     = null
}

variable "job_checkpoint_enabled" {
  description = "Whether to enable the automatic job snapshot function"
  type        = bool
  default     = false
}

variable "job_checkpoint_mode" {
  description = "The snapshot mode."
  type        = string
  default     = "exactly_once"
}

variable "job_checkpoint_interval" {
  description = "The snapshot interval in seconds"
  type        = number
  default     = 10
}

variable "job_obs_bucket" {
  description = "The OBS bucket used to save snapshots or logs"
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

variable "job_resume_max_num" {
  description = "The maximum number of retry times upon exceptions"
  type        = number
  default     = -1
}

variable "job_idle_state_retention" {
  description = "The idle state retention in seconds"
  type        = number
  default     = 1
}

variable "job_runtime_config" {
  description = "The custom optimization parameters when the Flink job is running"
  type        = map(string)
  default     = {}
}

variable "job_tags" {
  description = "The key/value pairs to associate with the Flink OpenSource SQL job"
  type        = map(string)
  default     = {}
}

variable "job_description" {
  description = "The description of the Flink OpenSource SQL job"
  type        = string
  default     = ""
}

resource "huaweicloud_dli_flinksql_job" "test" {
  name                   = var.job_name
  type                   = "flink_opensource_sql_job"
  run_mode               = "exclusive_cluster"
  queue_name             = huaweicloud_dli_queue.test.name
  sql                    = var.job_sql
  flink_version          = var.job_flink_version
  execution_agency_urn   = var.job_execution_agency_urn
  cu_number              = var.job_cu_number
  parallel_number        = var.job_parallel_number
  manager_cu_number      = var.job_manager_cu_number
  tm_cus                 = var.job_tm_cus
  tm_slot_num            = var.job_tm_slot_num
  checkpoint_enabled     = var.job_checkpoint_enabled
  checkpoint_mode        = var.job_checkpoint_mode
  checkpoint_interval    = var.job_checkpoint_interval
  obs_bucket             = var.job_obs_bucket
  log_enabled            = var.job_log_enabled
  smn_topic              = var.job_smn_topic
  restart_when_exception = var.job_restart_when_exception
  resume_max_num         = var.job_resume_max_num
  idle_state_retention   = var.job_idle_state_retention
  runtime_config         = var.job_runtime_config
  tags                   = var.job_tags
  description            = var.job_description

  depends_on = [huaweicloud_dli_queue.test]
}
```

**Parameter Description**:
- **name**: The name of the job, assigned by referencing the input variable job_name
- **type**: The type of the job, fixed to flink_opensource_sql_job
- **run_mode**: The run mode, fixed to exclusive_cluster
- **queue_name**: The name of the queue used to run the job, assigned by referencing the name of the queue created in the previous step
- **sql**: The Flink SQL statement that includes source, query and sink, assigned by referencing the input variable job_sql
- **flink_version**: The Flink version, assigned by referencing the input variable job_flink_version
- **execution_agency_urn**: The agency URN authorized to DLI, assigned by referencing the input variable job_execution_agency_urn
- **cu_number**: The number of CUs selected for the job, assigned by referencing the input variable job_cu_number
- **parallel_number**: The parallelism of the job, assigned by referencing the input variable job_parallel_number
- **manager_cu_number**: The number of CUs in the JobManager, assigned by referencing the input variable job_manager_cu_number
- **tm_cus**: The number of CUs for each Task Manager, assigned by referencing the input variable job_tm_cus
- **tm_slot_num**: The number of slots in each Task Manager, assigned by referencing the input variable job_tm_slot_num
- **checkpoint_enabled**: Whether to enable the automatic job snapshot function, assigned by referencing the input variable job_checkpoint_enabled
- **checkpoint_mode**: The snapshot mode, assigned by referencing the input variable job_checkpoint_mode
- **checkpoint_interval**: The snapshot interval in seconds, assigned by referencing the input variable job_checkpoint_interval
- **obs_bucket**: The OBS bucket used to save snapshots or logs, assigned by referencing the input variable job_obs_bucket
- **log_enabled**: Whether to enable uploading job logs to the OBS bucket, assigned by referencing the input variable job_log_enabled
- **smn_topic**: The SMN topic used to receive job failure notifications, assigned by referencing the input variable job_smn_topic
- **restart_when_exception**: Whether to enable the function of automatically restarting a job upon job exceptions, assigned by referencing the input variable job_restart_when_exception
- **resume_max_num**: The maximum number of retry times upon exceptions, assigned by referencing the input variable job_resume_max_num
- **idle_state_retention**: The idle state retention in seconds, assigned by referencing the input variable job_idle_state_retention
- **runtime_config**: The custom optimization parameters when the Flink job is running, assigned by referencing the input variable job_runtime_config
- **tags**: The key/value pairs to associate with the job, assigned by referencing the input variable job_tags
- **description**: The description of the job, assigned by referencing the input variable job_description

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
elastic_resource_pool_name = "tf_test_resource_pool"
elastic_resource_pool_cidr = "172.16.0.0/18"

elastic_resource_pool_label = {
  spec = "basic"
}

queue_name        = "tf_test_queue"
job_name          = "tf_test_flink_sql_job"
job_flink_version = "1.15"
job_sql           = <<-EOF
create table dataGenSource(
  user_id string,
  amount int
) with (
  'connector' = 'datagen',
  'rows-per-second' = '1',
  'fields.user_id.kind' = 'random',
  'fields.user_id.length' = '3'
);

create table printSink(
  user_id string,
  amount int
) with (
  'connector' = 'print'
);

insert into printSink select * from dataGenSource;
EOF
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="elastic_resource_pool_name=my-pool"`
2. Environment variables: `export TF_VAR_elastic_resource_pool_name=my-pool`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Flink OpenSource SQL job
4. Run `terraform show` to view the created Flink OpenSource SQL job

## Reference Information

- [Huawei Cloud Data Lake Insight Product Documentation](https://support.huaweicloud.com/dli/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DLI Flink OpenSource SQL Job](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dli/flink-opensource-sql-job)
