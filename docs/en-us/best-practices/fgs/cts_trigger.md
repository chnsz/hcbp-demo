# Deploy CTS Trigger

## Application Scenario

FunctionGraph's CTS trigger (Cloud Trace Service Trigger) is a trigger type based on the Cloud Trace Service (CTS) that can monitor and respond to Huawei Cloud resource operation events. Through CTS triggers, you can implement security auditing, compliance monitoring, automated response, event notification, and other functions.

CTS triggers are particularly suitable for scenarios that require real-time monitoring of cloud resource operations, security auditing, and automated operations, such as resource change monitoring, security event response, compliance checks, operation log analysis, etc. This best practice will introduce how to use Terraform to automatically deploy a FunctionGraph function with a CTS trigger.

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

variable "function_agency_name" {
  description = "The agency name of the FunctionGraph function"
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
# -*- coding:utf-8 -*-
'''
CTS trigger event:
{
  "cts":  {
        "time": "",
        "user": {
            "name": "userName",
            "id": "",
            "domain": {
                "name": "domainName",
                "id": ""
            }
        },
        "request": {},
        "response": {},
        "code": 204,
        "service_type": "FunctionGraph",
        "resource_type": "",
        "resource_name": "",
        "resource_id": {},
        "trace_name": "",
        "trace_type": "ConsoleAction",
        "record_time": "",
        "trace_id": "",
        "trace_status": "normal"
    }
}
'''
def handler (event, context):
    trace_name = event["cts"]["resource_name"]
    timeinfo = event["cts"]["time"]
    print(timeinfo+' '+trace_name)
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
  agency      = var.function_agency_name
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
- **agency**: Function agency name, assigned by referencing the input variable function_agency_name, used for function permissions to access other Huawei Cloud services
- **memory_size**: Function memory size (MB), assigned by referencing the input variable function_memory_size, default value is 128MB
- **timeout**: Function timeout (seconds), assigned by referencing the input variable function_timeout, default value is 10 seconds
- **runtime**: Function runtime environment, assigned by referencing the input variable function_runtime, default value is Python2.7
- **code_type**: Code type, set to "inline" for inline code
- **func_code**: Function source code, assigned by base64 encoding the input variable function_code
- **description**: Function description information, assigned by referencing the input variable function_description

### 3. Create FunctionGraph CTS Trigger

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a FunctionGraph CTS trigger resource:

```hcl
# Variable definitions for CTS trigger
variable "trigger_status" {
  description = "The status of the FunctionGraph CTS trigger"
  type        = string
  default     = "ACTIVE"
}

variable "trigger_name" {
  description = "The name of the FunctionGraph CTS trigger"
  type        = string
}

variable "trigger_operations" {
  description = "The operations to monitor for the FunctionGraph CTS trigger"
  type        = list(string)
  nullable    = false
}

# Create a FunctionGraph CTS trigger resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_fgs_function_trigger" "test" {
  function_urn = huaweicloud_fgs_function.test.urn
  type         = "CTS"
  status       = var.trigger_status
  event_data   = jsonencode({
    "name"       = var.trigger_name
    "operations" = var.trigger_operations
  })
}
```

**Parameter Description**:
- **function_urn**: URN of the FunctionGraph function associated with the trigger, assigned by referencing huaweicloud_fgs_function.test.urn
- **type**: Trigger type, set to "CTS" for CTS trigger
- **status**: Trigger status, assigned by referencing the input variable trigger_status, default value is "ACTIVE" for active status
- **event_data**: Trigger event data, in JSON format containing the following parameters:
  - **name**: Trigger name, assigned by referencing the input variable trigger_name
  - **operations**: List of operations to monitor, assigned by referencing the input variable trigger_operations, supports monitoring specific cloud service operations

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Function basic information
function_name        = "tf_test_cts_function"
function_agency_name = "function_all_trust"

# Trigger configuration
trigger_name         = "tf_test_cts_trigger"
trigger_operations   = [
  "FunctionGraph:Functions:createFunction"
]
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
3. After confirming the resource plan is correct, run `terraform apply` to start creating FunctionGraph function and CTS trigger
4. Run `terraform show` to view the created FunctionGraph function and CTS trigger

## Reference Information

- [Huawei Cloud FunctionGraph Product Documentation](https://support.huaweicloud.com/functiongraph/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [FunctionGraph Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/fgs)
