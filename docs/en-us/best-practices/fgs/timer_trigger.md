# Deploy Timer Trigger

## Application Scenario

FunctionGraph's timer trigger (Timer Trigger) is a trigger type based on time scheduling that supports both Cron expression and fixed interval scheduling methods. Through timer triggers, you can implement scheduled tasks, periodic data processing, scheduled backups, scheduled monitoring, and other functions.

Timer triggers are particularly suitable for tasks that need to be executed at fixed time intervals or specific time points, such as data cleanup, report generation, system monitoring, scheduled notifications, etc. This best practice will introduce how to use Terraform to automatically deploy a FunctionGraph function with a timer trigger.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

This best practice does not use data sources.

### Resources

- [FunctionGraph Function Resource (huaweicloud_fgs_function)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/fgs_function)
- [FunctionGraph Function Trigger Resource (huaweicloud_fgs_function_trigger)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/fgs_function_trigger)

### Resource/Data Source Dependencies

```
huaweicloud_fgs_function
    └── huaweicloud_fgs_function_trigger
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create FunctionGraph Function

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a FunctionGraph function resource:

```hcl
# Variable definitions for FunctionGraph resources
variable "function_name" {
  description = "The name of the FunctionGraph function"
  type        = string
}

variable "function_memory_size" {
  description = "The memory size of the function in MB"
  type        = number
  default     = 128
}

variable "function_timeout" {
  description = "The timeout of the function in seconds"
  type        = number
  default     = 10
}

variable "function_runtime" {
  description = "The runtime of the function"
  type        = string
  default     = "Python2.7"
}

variable "function_code" {
  description = "The source code of the function"
  type        = string
  default     = <<EOT
import json

def handler(event, context):
    print("FunctionGraph timer trigger executed!")
    print("Event:", json.dumps(event))
    print("Context:", json.dumps(context))
    return {
        'statusCode': 200,
        'body': json.dumps('Hello, FunctionGraph!')
    }
EOT
}

variable "function_description" {
  description = "The description of the function"
  type        = string
  default     = ""
}

# Create a FunctionGraph function resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_fgs_function" "test" {
  name        = var.function_name
  app         = "default"
  handler     = "index.handler"
  memory_size = var.function_memory_size
  timeout     = var.function_timeout
  runtime     = var.function_runtime
  code_type   = "inline"
  func_code   = base64encode(var.function_code)
  description = var.function_description
}
```

**Parameter Description**:
- **name**: FunctionGraph function name, assigned by referencing the input variable function_name
- **app**: Application name the function belongs to, set to "default" to use the default application
- **handler**: Function entry point, set to "index.handler" indicating the handler method is in the index.py file
- **memory_size**: Function memory size (MB), assigned by referencing the input variable function_memory_size, default value is 128MB
- **timeout**: Function timeout (seconds), assigned by referencing the input variable function_timeout, default value is 10 seconds
- **runtime**: Function runtime environment, assigned by referencing the input variable function_runtime, default value is Python2.7
- **code_type**: Code type, set to "inline" for inline code
- **func_code**: Function source code, assigned by base64 encoding the input variable function_code
- **description**: Function description information, assigned by referencing the input variable function_description

### 3. Create FunctionGraph Timer Trigger

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a FunctionGraph timer trigger resource:

```hcl
# Variable definitions for timer trigger
variable "trigger_status" {
  description = "The status of the FunctionGraph timer trigger"
  type        = string
  default     = "ACTIVE"
}

variable "trigger_name" {
  description = "The name of the FunctionGraph timer trigger"
  type        = string
}

variable "trigger_schedule_type" {
  description = "The schedule type of the FunctionGraph timer trigger"
  type        = string
  default     = "Cron"
}

variable "trigger_sync_execution" {
  description = "Whether to execute the function synchronously"
  type        = bool
  default     = false
}

variable "trigger_user_event" {
  description = "The user event description for the FunctionGraph timer trigger"
  type        = string
  default     = ""
}

variable "trigger_schedule" {
  description = "The schedule expression for the FunctionGraph timer trigger"
  type        = string
  default     = "@every 1h30m"
}

# Create a FunctionGraph timer trigger resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_fgs_function_trigger" "test" {
  function_urn = huaweicloud_fgs_function.test.urn
  type         = "TIMER"
  status       = var.trigger_status
  event_data   = jsonencode({
    "name"           = var.trigger_name
    "schedule_type"  = var.trigger_schedule_type
    "sync_execution" = var.trigger_sync_execution
    "user_event"     = var.trigger_user_event
    "schedule"       = var.trigger_schedule
  })
}
```

**Parameter Description**:
- **function_urn**: URN of the FunctionGraph function associated with the trigger, assigned by referencing huaweicloud_fgs_function.test.urn
- **type**: Trigger type, set to "TIMER" for timer trigger
- **status**: Trigger status, assigned by referencing the input variable trigger_status, default value is "ACTIVE" for active status
- **event_data**: Trigger event data, in JSON format containing the following parameters:
  - **name**: Trigger name, assigned by referencing the input variable trigger_name
  - **schedule_type**: Schedule type, assigned by referencing the input variable trigger_schedule_type, default value is "Cron" for Cron expression
  - **sync_execution**: Whether to execute synchronously, assigned by referencing the input variable trigger_sync_execution, default value is false for asynchronous execution
  - **user_event**: User event description, assigned by referencing the input variable trigger_user_event
  - **schedule**: Schedule expression, assigned by referencing the input variable trigger_schedule, supports Cron expression and fixed interval formats

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Function basic information
function_name        = "tf_test_timer_function"
function_description = "Created by Terraform for timer trigger example"

# Trigger configuration
trigger_name       = "tf_test_timer_cron"
trigger_user_event = "Timer trigger with Cron schedule type, triggered every three days"
trigger_schedule   = "@every 3d"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="function_name=my-function" -var="trigger_name=my-trigger"`
2. Environment variables: `export TF_VAR_function_name=my-function`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating FunctionGraph function and timer trigger
4. Run `terraform show` to view the created FunctionGraph function and timer trigger

## Reference Information

- [Huawei Cloud FunctionGraph Product Documentation](https://support.huaweicloud.com/functiongraph/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [FunctionGraph Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/fgs)
