# Deploy Global Connection Bandwidth

## Application Scenario

Cloud Connect (CC) global connection bandwidth is a global network bandwidth resource provided by the Cloud Connect service, used to provide bandwidth guarantee for network connections across regions and countries. By creating global connection bandwidth, you can uniformly manage and allocate network bandwidth resources globally, achieving flexible network bandwidth configuration and cross-border network connections. Automating global connection bandwidth creation through Terraform can ensure standardized and consistent bandwidth resource management, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create a Cloud Connect global connection bandwidth.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Cloud Connect Global Connection Bandwidth Resource (huaweicloud_cc_global_connection_bandwidth)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_global_connection_bandwidth)

### Resource/Data Source Dependencies

```
huaweicloud_cc_global_connection_bandwidth
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Cloud Connect Global Connection Bandwidth Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Cloud Connect global connection bandwidth resource:

```hcl
variable "global_connection_bandwidth_name" {
  description = "The name of the global connection bandwidth"
  type        = string
}

variable "bandwidth_type" {
  description = "The type of the global connection bandwidth"
  type        = string
}

variable "bordercross" {
  description = "Whether the global connection bandwidth crosses borders"
  type        = bool
}

variable "bandwidth_size" {
  description = "Bandwidth size of the global connection bandwidth"
  type        = number
}

variable "charge_mode" {
  description = "Billing option of the global connection bandwidth"
  type        = string
}

variable "global_connection_bandwidth_description" {
  description = "The description about the global connection bandwidth"
  type        = string
  default     = "Created by Terraform"
}

variable "global_connection_bandwidth_tags" {
  description = "The tags of the global connection bandwidth"
  type        = map(string)
  default     = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# Create Cloud Connect global connection bandwidth resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cc_global_connection_bandwidth" "test" {
  name        = var.global_connection_bandwidth_name
  type        = var.bandwidth_type
  bordercross = var.bordercross
  size        = var.bandwidth_size
  charge_mode = var.charge_mode
  description = var.global_connection_bandwidth_description

  tags = var.global_connection_bandwidth_tags
}
```

**Parameter Description**:
- **name**: The global connection bandwidth name, assigned by referencing the input variable global_connection_bandwidth_name
- **type**: The global connection bandwidth type, assigned by referencing the input variable bandwidth_type
- **bordercross**: Whether the global connection bandwidth crosses borders, assigned by referencing the input variable bordercross
- **size**: The bandwidth size of the global connection bandwidth, assigned by referencing the input variable bandwidth_size
- **charge_mode**: The billing option of the global connection bandwidth, assigned by referencing the input variable charge_mode
- **description**: The description of the global connection bandwidth, assigned by referencing the input variable global_connection_bandwidth_description, default value is "Created by Terraform"
- **tags**: The tags of the global connection bandwidth, assigned by referencing the input variable global_connection_bandwidth_tags, default value includes "Owner" and "Env" tags

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, the resource uses input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Cloud Connect Global Connection Bandwidth Configuration
global_connection_bandwidth_name = "tf_test_global_connection_bandwidth"
bandwidth_type                   = "bgp"
bordercross                      = true
bandwidth_size                   = 5
charge_mode                      = "bandwidth"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="global_connection_bandwidth_name=test-bandwidth" -var="bandwidth_size=10"`
2. Environment variables: `export TF_VAR_global_connection_bandwidth_name=test-bandwidth` and `export TF_VAR_bandwidth_size=10`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a Cloud Connect global connection bandwidth:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Cloud Connect global connection bandwidth
4. Run `terraform show` to view the details of the created Cloud Connect global connection bandwidth

## Reference Information

- [Huawei Cloud Cloud Connect Product Documentation](https://support.huaweicloud.com/cc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Global Connection Bandwidth](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cc/global-connection-bandwidth)
