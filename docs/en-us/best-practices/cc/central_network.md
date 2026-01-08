# Deploy Central Network

## Application Scenario

Cloud Connect (CC) central network is a core network management function provided by the Cloud Connect service, used to uniformly manage and connect multiple network instances. By creating a central network, you can build a centralized network architecture, achieving unified management and interconnection across regions and networks. Automating central network creation through Terraform can ensure standardized and consistent network resource management, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create a Cloud Connect central network.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Cloud Connect Central Network Resource (huaweicloud_cc_central_network)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_central_network)

### Resource/Data Source Dependencies

```
huaweicloud_cc_central_network
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Cloud Connect Central Network Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Cloud Connect central network resource:

```hcl
variable "central_network_name" {
  description = "The name of the central network"
  type        = string
}

variable "central_network_description" {
  description = "The description about the central network"
  type        = string
  default     = "Created by Terraform"
}

variable "enterprise_project_id" {
  description = "ID of the enterprise project that the central network belongs to"
  type        = string
  default     = "0"
}

variable "central_network_tags" {
  description = "The tags of the central network"
  type        = map(string)
  default     = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# Create Cloud Connect central network resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cc_central_network" "test" {
  name                  = var.central_network_name
  description           = var.central_network_description
  enterprise_project_id = var.enterprise_project_id

  tags = var.central_network_tags
}
```

**Parameter Description**:
- **name**: The central network name, assigned by referencing the input variable central_network_name
- **description**: The description of the central network, assigned by referencing the input variable central_network_description, default value is "Created by Terraform"
- **enterprise_project_id**: The enterprise project ID to which the central network belongs, assigned by referencing the input variable enterprise_project_id, default value is "0"
- **tags**: The tags of the central network, assigned by referencing the input variable central_network_tags, default value includes "Owner" and "Env" tags

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, the resource uses input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Cloud Connect Central Network Configuration
central_network_name  = "tf_test_central_network"
enterprise_project_id = "0"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="central_network_name=test-network" -var="enterprise_project_id=0"`
2. Environment variables: `export TF_VAR_central_network_name=test-network` and `export TF_VAR_enterprise_project_id=0`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a Cloud Connect central network:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Cloud Connect central network
4. Run `terraform show` to view the details of the created Cloud Connect central network

## Reference Information

- [Huawei Cloud Cloud Connect Product Documentation](https://support.huaweicloud.com/cc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Central Network](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cc/central-network)
