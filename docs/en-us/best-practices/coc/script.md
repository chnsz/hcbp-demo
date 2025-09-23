# Deploy Script

## Application Scenario

Cloud Operations Center (COC) is a one-stop operation and maintenance management platform provided by Huawei Cloud, offering enterprises a unified operation and maintenance management entry point. COC helps enterprises achieve automated operation and maintenance and intelligent management through script management, task scheduling, monitoring and alerting, improving operation and maintenance efficiency and quality.

Script management is one of the core functions of COC service, supporting the creation, management, and execution of various operation and maintenance scripts, including Shell scripts, Python scripts, PowerShell scripts, etc. Through script management, you can achieve version control, parameter management, risk level assessment, and other functions for operation and maintenance scripts, laying the foundation for subsequent script execution and task scheduling. This best practice will introduce how to use Terraform to automatically deploy COC scripts, including script creation and parameter configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

None

### Resources

- [COC Script Resource (huaweicloud_coc_script)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/coc_script)

### Resource/Data Source Dependencies

```
huaweicloud_coc_script.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create COC Script

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a COC script resource:

```hcl
variable "script_name" {
  description = "Script name"
  type        = string
}

variable "script_description" {
  description = "Script description"
  type        = string
}

variable "script_risk_level" {
  description = "Script risk level"
  type        = string
}

variable "script_version" {
  description = "Script version"
  type        = string
}

variable "script_type" {
  description = "Script type"
  type        = string
}

variable "script_content" {
  description = "Script content"
  type        = string
}

variable "script_parameters" {
  description = "Script parameter list"
  type = list(object({
    name        = string
    value       = string
    description = string
    sensitive   = optional(bool)
  }))

  nullable = false
}

# Create a COC script resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_coc_script" "test" {
  name        = var.script_name
  description = var.script_description
  risk_level  = var.script_risk_level
  version     = var.script_version
  type        = var.script_type
  content     = var.script_content

  dynamic "parameters" {
    for_each = var.script_parameters

    content {
      name        = parameters.value.name
      value       = parameters.value.value
      description = parameters.value.description
      sensitive   = parameters.value.sensitive
    }
  }
}
```

**Parameter Description**:
- **name**: Script name, assigned by referencing the input variable script_name
- **description**: Script description, assigned by referencing the input variable script_description
- **risk_level**: Script risk level, assigned by referencing the input variable script_risk_level
- **version**: Script version, assigned by referencing the input variable script_version
- **type**: Script type, assigned by referencing the input variable script_type
- **content**: Script content, assigned by referencing the input variable script_content
- **parameters.name**: Parameter name, assigned by referencing the name field in the script parameter list
- **parameters.value**: Parameter value, assigned by referencing the value field in the script parameter list
- **parameters.description**: Parameter description, assigned by referencing the description field in the script parameter list
- **parameters.sensitive**: Whether the parameter is sensitive, assigned by referencing the sensitive field in the script parameter list

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Script configuration
script_name        = "tf_coc_script"
script_description = "Created by terraform script"
script_risk_level  = "LOW"
script_version     = "1.0.0"
script_type        = "SHELL"
script_content     = <<EOF
#! /bin/bash
echo "hello world!"
EOF

# Script parameter configuration
script_parameters = [
  {
    name        = "name"
    value       = "world"
    description = "the first parameter"
  },
  {
    name        = "company"
    value       = "Huawei"
    description = "the second parameter"
    sensitive   = true
  }
]
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="script_name=my-script" -var="script_type=SHELL"`
2. Environment variables: `export TF_VAR_script_name=my-script`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the COC script
4. Run `terraform show` to view the details of the created COC script

## Reference Information

- [Huawei Cloud COC Product Documentation](https://support.huaweicloud.com/coc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [COC Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/coc)
