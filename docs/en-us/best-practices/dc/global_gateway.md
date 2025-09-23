# Deploy Global Gateway

## Application Scenario

Direct Connect (DC) is a high-performance, low-latency, secure, and reliable dedicated line access service provided by Huawei Cloud, offering enterprises dedicated network connections from local data centers to Huawei Cloud. Direct Connect service supports multiple access methods, including physical dedicated lines and virtual dedicated lines, meeting network connection requirements for different scales and scenarios.

Global gateway is an advanced component in Direct Connect service, used to establish cross-region, cross-network global dedicated line connections. Through global gateways, you can achieve unified access management for multiple regions and networks, supporting BGP routing protocol, meeting the needs of large enterprises and complex network environments. This best practice will introduce how to use Terraform to automatically deploy DC global gateways, including gateway creation, BGP configuration, and tag management.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

None

### Resources

- [DC Global Gateway Resource (huaweicloud_dc_global_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dc_global_gateway)

### Resource/Data Source Dependencies

```
huaweicloud_dc_global_gateway.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create DC Global Gateway

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DC global gateway resource:

```hcl
variable "global_gateway_name" {
  description = "Global gateway name"
  type        = string
}

variable "global_gateway_description" {
  description = "Global gateway description"
  type        = string
  default     = "Created by Terraform"
}

variable "address_family" {
  description = "Global gateway IP address family"
  type        = string
  default     = "ipv4"
}

variable "bgp_asn" {
  description = "Global gateway BGP ASN"
  type        = number
}

variable "enterprise_project_id" {
  description = "Enterprise project ID to which the global gateway belongs"
  type        = string
  default     = "0"
}

variable "global_gateway_tags" {
  description = "Global gateway tags"
  type        = map(string)
  default = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# Create a DC global gateway resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_dc_global_gateway" "test" {
  name                  = var.global_gateway_name
  description           = var.global_gateway_description
  address_family        = var.address_family
  bgp_asn               = var.bgp_asn
  enterprise_project_id = var.enterprise_project_id

  tags = var.global_gateway_tags
}
```

**Parameter Description**:
- **name**: Global gateway name, assigned by referencing the input variable global_gateway_name
- **description**: Global gateway description, assigned by referencing the input variable global_gateway_description
- **address_family**: IP address family, assigned by referencing the input variable address_family, supports "ipv4" and "ipv6"
- **bgp_asn**: BGP ASN, assigned by referencing the input variable bgp_asn, used for BGP routing protocol configuration
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **tags**: Tags, assigned by referencing the input variable global_gateway_tags, used for resource classification and management

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Global gateway configuration
global_gateway_name        = "tf_test_global_gateway"
global_gateway_description = "Created by Terraform"
address_family             = "ipv4"
bgp_asn                    = 65000
enterprise_project_id      = "0"

# Tag configuration
global_gateway_tags = {
  "Owner"     = "terraform"
  "Env"       = "test"
  "Project"   = "dc-demo"
  "CostCenter" = "IT"
}
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="global_gateway_name=my-gateway" -var="bgp_asn=65000"`
2. Environment variables: `export TF_VAR_global_gateway_name=my-gateway`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DC global gateway
4. Run `terraform show` to view the details of the created DC global gateway

## Reference Information

- [Huawei Cloud DC Product Documentation](https://support.huaweicloud.com/dc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DC Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dc)
