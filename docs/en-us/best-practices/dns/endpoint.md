# Deploy Endpoint

## Application Scenario

Domain Name Service (DNS) is a highly available, high-performance domain name resolution service provided by Huawei Cloud, supporting both public and private domain name resolution. DNS service provides intelligent resolution, load balancing, health check, and other functions, helping users achieve intelligent scheduling and failover of domain names.

DNS endpoints are network access points in DNS service, used to provide DNS resolution services in VPC networks. Through endpoints, enterprises can deploy DNS services in private networks, implementing internal domain name resolution, private DNS forwarding, and other functions. Endpoints support both inbound and outbound directions, meeting DNS resolution requirements for different scenarios. This best practice will introduce how to use Terraform to automatically deploy DNS endpoints, including VPC creation, subnet configuration, and endpoint deployment.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [DNS Endpoint Resource (huaweicloud_dns_endpoint)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_endpoint)

### Resource/Data Source Dependencies

```
huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_dns_endpoint.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create VPC

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  description = "VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "192.168.0.0/16"
}

# Create a VPC resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable vpc_cidr

### 3. Create VPC Subnet

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "Subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "Subnet CIDR block"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "Subnet gateway IP"
  type        = string
  default     = ""
}

# Create a VPC subnet resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: VPC ID, assigned by referencing the VPC resource (huaweicloud_vpc.test) ID
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block, prioritizes using input variable, automatically calculated if empty
- **gateway_ip**: Gateway IP, prioritizes using input variable, automatically calculated if empty

### 4. Create DNS Endpoint

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DNS endpoint resource:

```hcl
variable "dns_endpoint_name" {
  description = "DNS endpoint name"
  type        = string
}

variable "dns_endpoint_direction" {
  description = "DNS endpoint direction"
  type        = string
  default     = "inbound"
}

# Create a DNS endpoint resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_dns_endpoint" "test" {
  name      = var.dns_endpoint_name
  direction = var.dns_endpoint_direction

  ip_addresses {
    subnet_id = huaweicloud_vpc_subnet.test.id
  }

  ip_addresses {
    subnet_id = huaweicloud_vpc_subnet.test.id
  }
}
```

**Parameter Description**:
- **name**: Endpoint name, assigned by referencing the input variable dns_endpoint_name
- **direction**: Endpoint direction, assigned by referencing the input variable dns_endpoint_direction, defaults to "inbound"
- **ip_addresses.subnet_id**: IP address subnet ID, assigned by referencing the VPC subnet resource (huaweicloud_vpc_subnet.test) ID

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Network configuration
vpc_name    = "tf_test_dns_endpoint"
subnet_name = "tf_test_dns_endpoint"

# DNS endpoint configuration
dns_endpoint_name = "tf_test_dns_endpoint"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="dns_endpoint_name=my-endpoint"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DNS endpoint
4. Run `terraform show` to view the details of the created DNS endpoint

## Reference Information

- [Huawei Cloud DNS Product Documentation](https://support.huaweicloud.com/dns/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DNS Endpoint](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dns/endpoint)
