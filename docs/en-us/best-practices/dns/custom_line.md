# Deploy Custom Line

## Application Scenario

Domain Name Service (DNS) is a highly available, high-performance domain name resolution service provided by Huawei Cloud, supporting both public and private domain name resolution. DNS service provides intelligent resolution, load balancing, health check, and other functions, helping users achieve intelligent scheduling and failover of domain names.

Custom lines are advanced features in DNS service that allow users to create custom resolution lines based on specific IP address segments, achieving more fine-grained traffic scheduling and resolution control. Through custom lines, enterprises can provide different resolution results for different user groups based on factors such as user geographic location, network operator, IP address segment, etc., implementing intelligent resolution and load balancing. This best practice will introduce how to use Terraform to automatically deploy DNS custom lines, including line creation and IP address segment configuration.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DNS Custom Line Resource (huaweicloud_dns_custom_line)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_custom_line)

### Resource/Data Source Dependencies

```
No dependencies
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create DNS Custom Line

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DNS custom line resource:

```hcl
variable "dns_custom_line_name" {
  description = "Custom line name"
  type        = string
}

variable "dns_custom_line_ip_segments" {
  description = "IP address segments"
  type        = list(string)
}

# Create a DNS custom line resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_dns_custom_line" "test" {
  name        = var.dns_custom_line_name
  ip_segments = var.dns_custom_line_ip_segments
}
```

**Parameter Description**:
- **name**: Custom line name, assigned by referencing the input variable dns_custom_line_name
- **ip_segments**: IP address segment list, assigned by referencing the input variable dns_custom_line_ip_segments

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# DNS custom line configuration
dns_custom_line_name        = "your_custom_line_name"
dns_custom_line_ip_segments = ["100.100.100.102-100.100.100.102", "100.100.100.101-100.100.100.101"]
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="dns_custom_line_name=my-line" -var="dns_custom_line_ip_segments=[\"192.168.1.1-192.168.1.10\"]"`
2. Environment variables: `export TF_VAR_dns_custom_line_name=my-line`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DNS custom line
4. Run `terraform show` to view the details of the created DNS custom line

## Reference Information

- [Huawei Cloud DNS Product Documentation](https://support.huaweicloud.com/dns/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DNS Custom Line](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dns/custom-line)
