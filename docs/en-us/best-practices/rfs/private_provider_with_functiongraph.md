# Deploy Private Provider with FunctionGraph Backend

## Application Scenario

Resource Formation Service (RFS) is an Infrastructure as Code (IaC) service provided by Huawei Cloud, supporting the definition, orchestration, and automated deployment of cloud resources in a template-based manner. A private provider is an extension capability of RFS that allows users to register custom execution logic as a provider, so that custom resource types can be invoked in templates.

FunctionGraph is an event-driven serverless computing service that allows you to run code without managing servers. Using a FunctionGraph function as the execution backend of a private provider encapsulates custom resource operation logic into a function, which RFS can invoke on demand during orchestration to achieve flexible resource extension.

This best practice will introduce how to use Terraform to automatically deploy an RFS private provider with a FunctionGraph backend, including creating a FunctionGraph function, creating a private provider associated with the function, and publishing a new version for the private provider.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [FunctionGraph Function (huaweicloud_fgs_function)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/fgs_function)
- [RFS Private Provider (huaweicloud_rfs_private_provider)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_private_provider)
- [RFS Private Provider Version (huaweicloud_rfs_private_provider_version)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_private_provider_version)

### Resource/Data Source Dependencies

```
huaweicloud_fgs_function
    ├── huaweicloud_rfs_private_provider
    │       └── huaweicloud_rfs_private_provider_version
    └── huaweicloud_rfs_private_provider_version
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a FunctionGraph Function

Add the following script in the TF file (such as main.tf) to create the FunctionGraph function that serves as the execution backend of the private provider:

```hcl
# Create the FunctionGraph function resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "function_name" {
  description = "The name of the FunctionGraph function"
  type        = string
}

variable "function_app" {
  description = "The group name of the FunctionGraph function"
  type        = string
  default     = "default"
}

variable "function_handler" {
  description = "The handler of the FunctionGraph function"
  type        = string
  default     = "index.handler"
}

variable "function_memory_size" {
  description = "The memory size of the FunctionGraph function in MB"
  type        = number
  default     = 128
}

variable "function_timeout" {
  description = "The timeout of the FunctionGraph function in seconds"
  type        = number
  default     = 3
}

variable "function_runtime" {
  description = "The runtime of the FunctionGraph function"
  type        = string
  default     = "Node.js12.13"
}

variable "function_code" {
  description = "The inline code content of the FunctionGraph function"
  type        = string
}

resource "huaweicloud_fgs_function" "test" {
  name        = var.function_name
  app         = var.function_app
  handler     = var.function_handler
  memory_size = var.function_memory_size
  timeout     = var.function_timeout
  code_type   = "inline"
  runtime     = var.function_runtime
  func_code   = base64encode(var.function_code)
}
```

**Parameter Description**:
- **name**: The function name, assigned by referencing the input variable function_name
- **app**: The application group the function belongs to, assigned by referencing the input variable function_app, defaulting to default
- **handler**: The function handler, assigned by referencing the input variable function_handler, defaulting to index.handler
- **memory_size**: The function memory size in MB, assigned by referencing the input variable function_memory_size, defaulting to 128
- **timeout**: The function timeout in seconds, assigned by referencing the input variable function_timeout, defaulting to 3
- **code_type**: The function code type, fixed to inline here, indicating inline code is used
- **runtime**: The function runtime, assigned by referencing the input variable function_runtime, defaulting to Node.js12.13
- **func_code**: The inline function code content, assigned by referencing the input variable function_code and encoded with base64encode

### 3. Create an RFS Private Provider

Add the following script in the TF file (such as main.tf) to create the RFS private provider associated with the FunctionGraph function above:

```hcl
# Create the RFS private provider resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "private_provider_name" {
  description = "The name of the RFS private provider"
  type        = string
}

variable "private_provider_description" {
  description = "The description of the RFS private provider"
  type        = string
  default     = ""
}

variable "private_provider_version" {
  description = "The initial version number of the RFS private provider"
  type        = string
  default     = "1.0.0"
}

variable "private_provider_version_description" {
  description = "The initial version description of the RFS private provider"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_private_provider" "test" {
  provider_name        = var.private_provider_name
  function_graph_urn   = huaweicloud_fgs_function.test.urn
  provider_description = var.private_provider_description
  provider_version     = var.private_provider_version
  version_description  = var.private_provider_version_description
}
```

**Parameter Description**:
- **provider_name**: The private provider name, assigned by referencing the input variable private_provider_name; only lowercase letters, digits, and hyphens (-) are allowed, and it must be unique within its domain and region
- **function_graph_urn**: The URN of the FunctionGraph function used as the execution backend, assigned by referencing huaweicloud_fgs_function.test.urn created in the previous step; this parameter is non-updatable and changing it will recreate the private provider
- **provider_description**: The private provider description, assigned by referencing the input variable private_provider_description
- **provider_version**: The initial version number of the private provider, assigned by referencing the input variable private_provider_version, defaulting to 1.0.0; this parameter is non-updatable
- **version_description**: The initial version description, assigned by referencing the input variable private_provider_version_description; this parameter is non-updatable

### 4. Create an RFS Private Provider Version

Add the following script in the TF file (such as main.tf) to publish a new version for the private provider:

```hcl
# Create the RFS private provider version resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "provider_version_number" {
  description = "The version number of the RFS private provider version"
  type        = string
  default     = "2.0.0"
}

variable "provider_version_description" {
  description = "The description of the RFS private provider version"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_private_provider_version" "test" {
  provider_name       = huaweicloud_rfs_private_provider.test.provider_name
  provider_version    = var.provider_version_number
  function_graph_urn  = huaweicloud_fgs_function.test.urn
  version_description = var.provider_version_description
}
```

**Parameter Description**:
- **provider_name**: The private provider name, assigned by referencing huaweicloud_rfs_private_provider.test.provider_name created in the previous step
- **provider_version**: The new version number, assigned by referencing the input variable provider_version_number, defaulting to 2.0.0
- **function_graph_urn**: The URN of the FunctionGraph function used as the execution backend, assigned by referencing huaweicloud_fgs_function.test.urn
- **version_description**: The version description, assigned by referencing the input variable provider_version_description

> Note: All arguments of the private provider version are non-updatable, and changing them will recreate the version.

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
function_name         = "tf-test-function"
function_code         = <<EOT
exports.handler = async (event, context) => {
    const result =
    {
        'statusCode': 200,
        'headers':
        {
            'Content-Type': 'application/json'
        },
        'isBase64Encoded': false,
        'body': JSON.stringify(event)
    }
    return result
}
EOT
private_provider_name = "tf-test-provider"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="private_provider_name=my-provider"`
2. Environment variables: `export TF_VAR_private_provider_name=my-provider`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the private provider with a FunctionGraph backend
4. Run `terraform show` to view the created private provider with a FunctionGraph backend

## Reference Information

- [Huawei Cloud Resource Formation Service (RFS) Product Documentation](https://support.huaweicloud.com/rfs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For RFS Private Provider with FunctionGraph Backend](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rfs/private-provider-with-functiongraph)
