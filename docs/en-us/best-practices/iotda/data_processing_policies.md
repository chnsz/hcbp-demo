# Deploy Data Flow Control and Backlog Policies

## Application Scenario

IoT Device Access (IoTDA) is a device access and management service provided by Huawei Cloud, supporting massive device connections, data collection, and data forwarding. In device data forwarding scenarios, if a forwarding target becomes abnormal or the downstream processing capability is insufficient, the number of forwarding requests may surge or a large amount of data may accumulate, affecting the normal running of other services under the tenant.

This best practice will introduce how to use Terraform to automatically deploy an IoTDA data flow control policy and a data backlog policy, limiting the tenant-level data forwarding TPS and controlling the backlog size and backlog time of forwarded data, so as to ensure the stability and reliability of the data forwarding link.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Data Flow Control Policy (huaweicloud_iotda_data_flow_control_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/iotda_data_flow_control_policy)
- [Data Backlog Policy (huaweicloud_iotda_data_backlog_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/iotda_data_backlog_policy)

### Resource/Data Source Dependencies

```
huaweicloud_iotda_data_flow_control_policy

huaweicloud_iotda_data_backlog_policy
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Data Flow Control Policy

Add the following script in the TF file (such as main.tf) to create a data flow control policy:

```hcl
# Create a data flow control policy resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "flow_control_policy_name" {
  description = "The name of the data flow control policy"
  type        = string
}

variable "flow_control_policy_description" {
  description = "The description of the data flow control policy"
  type        = string
  default     = ""
}

variable "flow_control_policy_limit" {
  description = "The flow control limit of the policy in tps"
  type        = number
}

resource "huaweicloud_iotda_data_flow_control_policy" "test" {
  name        = var.flow_control_policy_name
  description = var.flow_control_policy_description
  scope       = "USER"
  limit       = var.flow_control_policy_limit
}
```

**Parameter Description**:
- **name**: The name of the data flow control policy, assigned by referencing the input variable flow_control_policy_name. The name must not exceed 256 characters, only Chinese characters, letters, digits and the following characters are allowed: `_?'#().,&%@!-`, spaces are not allowed
- **description**: The description of the data flow control policy, assigned by referencing the input variable flow_control_policy_description. The description must not exceed 256 characters
- **scope**: The scope of the data flow control policy, fixed to `USER` in this practice, which means tenant-level flow control, so the scope_value argument is not required
- **limit**: The flow control limit of the policy in tps, assigned by referencing the input variable flow_control_policy_limit. The value ranges from 1 to 1,000, and the default value is 1,000

### 3. Create a Data Backlog Policy

Add the following script in the TF file (such as main.tf) to create a data backlog policy:

```hcl
# Create a data backlog policy resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "backlog_policy_name" {
  description = "The name of the data backlog policy"
  type        = string
}

variable "backlog_policy_description" {
  description = "The description of the data backlog policy"
  type        = string
  default     = ""
}

variable "backlog_policy_size" {
  description = "The size of data backlog in bytes"
  type        = string
}

variable "backlog_policy_time" {
  description = "The data backlog time in seconds"
  type        = string
}

resource "huaweicloud_iotda_data_backlog_policy" "test" {
  name         = var.backlog_policy_name
  description  = var.backlog_policy_description
  backlog_size = var.backlog_policy_size
  backlog_time = var.backlog_policy_time
}
```

**Parameter Description**:
- **name**: The name of the data backlog policy, assigned by referencing the input variable backlog_policy_name. The name must not exceed 256 characters, only Chinese characters, letters, digits and the following characters are allowed: `_?'#().,&%@!-`, spaces are not allowed
- **description**: The description of the data backlog policy, assigned by referencing the input variable backlog_policy_description. The description must not exceed 256 characters
- **backlog_size**: The size of data backlog in bytes, assigned by referencing the input variable backlog_policy_size. The value ranges from 0 to 1,073,741,823, and 0 means no backlog
- **backlog_time**: The data backlog time in seconds, assigned by referencing the input variable backlog_policy_time. The value ranges from 0 to 86,399, and 0 means no backlog. When both dimensions are configured, the dimension that reaches the threshold first shall prevail

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Data flow control policy parameters
flow_control_policy_name        = "tf_test_iotda_flow_control_policy"
flow_control_policy_description = "Limit-the-data-forwarding-tps-of-the-tenant"
flow_control_policy_limit       = 500

# Data backlog policy parameters
backlog_policy_name        = "tf_test_iotda_backlog_policy"
backlog_policy_description = "Control-the-size-and-time-of-forwarded-data-backlog"
backlog_policy_size        = "524288000"
backlog_policy_time        = "3600"

# The HTTPS application access address of the IoTDA instance
iotda_access_address = "https://779f0f0dd5.st1.iotda-app.cn-north-4.myhuaweicloud.com"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="flow_control_policy_name=my-policy"`
2. Environment variables: `export TF_VAR_flow_control_policy_name=my-policy`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the data flow control policy and the data backlog policy
4. Run `terraform show` to view the created data flow control policy and data backlog policy

> Note: The data flow control policy and the data backlog policy are only supported on IoTDA standard and enterprise edition instances. The instance access address must be specified through the `endpoints.iotda` argument of the provider block, and the provider version must be 1.71.0 or later.

## Reference Information

- [Huawei Cloud IoTDA Product Documentation](https://support.huaweicloud.com/iotda/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For IoTDA Data Flow Control and Backlog Policies](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/iotda/data-processing-policies)
