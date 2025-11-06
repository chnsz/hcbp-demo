# Deploy Basic Network

## Application Scenario

Virtual Private Cloud (VPC) is a logically isolated network space that users can customize and manage on Huawei Cloud. Through VPC, users can flexibly divide subnets, configure routes and security policies, implementing secure isolation and efficient management of cloud resources. This best practice will introduce how to use Terraform to automatically deploy a basic VPC and its subnets.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)

### Resource/Data Source Dependencies

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create VPC Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "172.16.0.0/16"
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the VPC belongs"
  type        = string
  default     = null
}

# Create VPC resource
resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable vpc_cidr, default value is "172.16.0.0/16"
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is null

### 3. Create VPC Subnet Resource

Add the following script to the TF file to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "172.16.10.0/24"
}

variable "subnet_gateway" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = "172.16.10.1"
}

variable "dns_list" {
  description = "The list of DNS server IP addresses"
  type        = list(string)
  default     = null
}

# Create VPC subnet resource
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway
  dns_list   = var.dns_list
}
```

**Parameter Description**:
- **vpc_id**: VPC ID that the subnet belongs to, referencing the ID of the previously created VPC resource
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block, assigned by referencing the input variable subnet_cidr, default value is "172.16.10.0/24"
- **gateway_ip**: Subnet gateway IP, assigned by referencing the input variable subnet_gateway, default value is "172.16.10.1"
- **dns_list**: List of DNS server IP addresses for the subnet, assigned by referencing the input variable dns_list, default value is null

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC configuration
vpc_name              = "tf_test_vpc"
vpc_cidr              = "172.16.0.0/16"
enterprise_project_id = null

# Subnet configuration
subnet_name    = "tf_test_subnet"
subnet_cidr    = "172.16.10.0/24"
subnet_gateway = "172.16.10.1"
dns_list       = ["114.114.114.114", "8.8.8.8"]
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating VPC and subnets
4. Run `terraform show` to view the created VPC and subnet details

## Reference Information

- [Huawei Cloud VPC Product Documentation](https://support.huaweicloud.com/vpc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For VPC Basic Network](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpc/basic)
