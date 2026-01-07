# Deploy Bandwidth Package

## Application Scenario

Cloud Connect (CC) bandwidth package is a bandwidth resource management function provided by the Cloud Connect service, used to provide bandwidth resources for Cloud Connect instances. By creating bandwidth packages, you can uniformly manage and allocate bandwidth resources across regions and networks, achieving flexible network bandwidth configuration. Automating bandwidth package creation through Terraform can ensure standardized and consistent bandwidth resource management, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create a Cloud Connect bandwidth package.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Cloud Connect Bandwidth Package Resource (huaweicloud_cc_bandwidth_package)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_bandwidth_package)

### Resource/Data Source Dependencies

```
huaweicloud_cc_bandwidth_package
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Cloud Connect Bandwidth Package Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Cloud Connect bandwidth package resource:

```hcl
variable "bandwidth_package_name" {
  description = "The name of the bandwidth package"
  type        = string
}

variable "local_area_id" {
  description = "The local area ID"
  type        = string
}

variable "remote_area_id" {
  description = "The remote area ID"
  type        = string
}

variable "charge_mode" {
  description = "Billing option of the bandwidth package"
  type        = string
}

variable "billing_mode" {
  description = "Billing mode of the bandwidth package"
  type        = string
}

variable "bandwidth" {
  description = "Bandwidth in the bandwidth package"
  type        = number
}

variable "bandwidth_package_description" {
  description = "The description about the bandwidth package"
  type        = string
  default     = "Created by Terraform"
}

variable "enterprise_project_id" {
  description = "ID of the enterprise project that the bandwidth package belongs to"
  type        = string
  default     = "0"
}

variable "bandwidth_package_tags" {
  description = "The tags of the bandwidth package"
  type        = map(string)
  default     = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# Create Cloud Connect bandwidth package resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cc_bandwidth_package" "test" {
  name                  = var.bandwidth_package_name
  local_area_id         = var.local_area_id
  remote_area_id        = var.remote_area_id
  charge_mode           = var.charge_mode
  billing_mode          = var.billing_mode
  bandwidth             = var.bandwidth
  description           = var.bandwidth_package_description
  enterprise_project_id = var.enterprise_project_id

  tags = var.bandwidth_package_tags
}
```

**Parameter Description**:
- **name**: The bandwidth package name, assigned by referencing the input variable bandwidth_package_name
- **local_area_id**: The local area ID, assigned by referencing the input variable local_area_id
- **remote_area_id**: The remote area ID, assigned by referencing the input variable remote_area_id
- **charge_mode**: The billing option of the bandwidth package, assigned by referencing the input variable charge_mode
- **billing_mode**: The billing mode of the bandwidth package, assigned by referencing the input variable billing_mode
- **bandwidth**: The bandwidth value in the bandwidth package, assigned by referencing the input variable bandwidth
- **description**: The description of the bandwidth package, assigned by referencing the input variable bandwidth_package_description, default value is "Created by Terraform"
- **enterprise_project_id**: The enterprise project ID to which the bandwidth package belongs, assigned by referencing the input variable enterprise_project_id, default value is "0"
- **tags**: The tags of the bandwidth package, assigned by referencing the input variable bandwidth_package_tags, default value includes "Owner" and "Env" tags

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, the resource uses input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Cloud Connect Bandwidth Package Configuration
bandwidth_package_name = "tf_test_bandwidth_package"
local_area_id          = "Chinese-Mainland"
remote_area_id         = "Chinese-Mainland"
charge_mode            = "bandwidth"
billing_mode           = "prepaid"
bandwidth              = 5
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="bandwidth_package_name=test-package" -var="bandwidth=10"`
2. Environment variables: `export TF_VAR_bandwidth_package_name=test-package` and `export TF_VAR_bandwidth=10`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a Cloud Connect bandwidth package:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Cloud Connect bandwidth package
4. Run `terraform show` to view the details of the created Cloud Connect bandwidth package

## Reference Information

- [Huawei Cloud Cloud Connect Product Documentation](https://support.huaweicloud.com/cc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Bandwidth Package](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cc/bandwidth-package)
