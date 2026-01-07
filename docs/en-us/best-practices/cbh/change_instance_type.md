# Deploy Change Instance Type

## Application Scenario

Cloud Bastion Host (CBH) instance type modification is an important operation and maintenance function provided by the CBH service. When you need to adjust the computing capability, storage space, or network performance of a cloud bastion host instance, you can modify the instance type to meet changing business requirements. Automating instance type modification operations through Terraform can ensure standardized and consistent resource configuration, improving operational efficiency. This best practice will introduce how to use Terraform to automatically modify the type of a single-node CBH instance.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Cloud Bastion Host Change Instance Type Resource (huaweicloud_cbh_change_instance_type)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cbh_change_instance_type)

### Resource/Data Source Dependencies

```
huaweicloud_cbh_change_instance_type
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Cloud Bastion Host Change Instance Type Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a cloud bastion host change instance type resource:

```hcl
variable "server_id" {
  description = "The ID of the single node CBH instance to change type"
  type        = string
}

variable "availability_zone" {
  description = "The availability zone of the single-node CBH instance to change type"
  type        = string
  default     = ""
}

# Create cloud bastion host change instance type resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cbh_change_instance_type" "test" {
  server_id         = var.server_id
  availability_zone = var.availability_zone
}
```

**Parameter Description**:
- **server_id**: The ID of the single-node CBH instance to change type, assigned by referencing the input variable server_id
- **availability_zone**: The availability zone of the single-node CBH instance, assigned by referencing the input variable availability_zone, default value is an empty string

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, the resource uses input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Cloud Bastion Host Change Instance Type Configuration
server_id = "your_single_node_server_id"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="server_id=your-server-id" -var="availability_zone=your-az"`
2. Environment variables: `export TF_VAR_server_id=your-server-id` and `export TF_VAR_availability_zone=your-az`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to modify the single-node CBH instance type:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start modifying the single-node CBH instance type
4. Run `terraform show` to view the details of the created cloud bastion host change instance type resource

> Note: It takes about 15 minutes to modify the instance type of the single-node CBH instance, please be patient.

## Reference Information

- [Huawei Cloud Cloud Bastion Host Product Documentation](https://support.huaweicloud.com/cbh/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Change Instance Type](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cbh/change-instance-type)
