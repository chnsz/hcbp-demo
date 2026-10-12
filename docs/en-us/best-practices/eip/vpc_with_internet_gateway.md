# Deploy VPC Internet Gateway

## Application Scenario

A VPC internet gateway is a key component of Huawei Cloud Virtual Private Cloud (VPC) that connects a VPC to the public network, providing cloud resources in the VPC with the ability to access the public network and be accessed from it. In scenarios such as cross-region IPv4 networking (GEIP), the internet gateway is a prerequisite for public network interconnection of a VPC. Only after the internet gateway is created can resources in the VPC communicate over the public network.

This best practice will introduce how to use Terraform to automatically deploy a VPC internet gateway, including VPC creation, subnet creation, and the creation and route configuration of the internet gateway.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [VPC Internet Gateway (huaweicloud_vpc_internet_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_internet_gateway)

### Resource/Data Source Dependencies

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_vpc_internet_gateway
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Virtual Private Cloud

Add the following script in the TF file (such as main.tf) to create a VPC:

```hcl
# Create a virtual private cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr, with a default value of `192.168.0.0/16`

### 3. Create a VPC Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create a VPC subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the resource huaweicloud_vpc.test
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet, assigned by referencing the input variable subnet_cidr; when the variable is an empty string, it is automatically divided based on the VPC CIDR block
- **gateway_ip**: The gateway IP address of the subnet, assigned by referencing the input variable subnet_gateway_ip; when the variable is an empty string, it is automatically calculated based on the subnet CIDR block

### 4. Create a VPC Internet Gateway

Add the following script in the TF file (such as main.tf) to create an internet gateway:

```hcl
# Create a VPC internet gateway resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "internet_gateway_name" {
  description = "The name of the VPC internet gateway"
  type        = string
}

variable "internet_gateway_add_route" {
  description = "Whether to add a default route pointing to the internet gateway"
  type        = bool
  default     = true
}

variable "internet_gateway_ipv6_enabled" {
  description = "Whether to enable IPv6 for the internet gateway"
  type        = bool
  default     = false
}

resource "huaweicloud_vpc_internet_gateway" "test" {
  vpc_id      = huaweicloud_vpc.test.id
  subnet_id   = huaweicloud_vpc_subnet.test.id
  name        = var.internet_gateway_name
  add_route   = var.internet_gateway_add_route
  enable_ipv6 = var.internet_gateway_ipv6_enabled
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the internet gateway belongs, assigned by referencing the ID of the resource huaweicloud_vpc.test
- **subnet_id**: The ID of the subnet associated with the internet gateway, assigned by referencing the ID of the resource huaweicloud_vpc_subnet.test
- **name**: The internet gateway name, assigned by referencing the input variable internet_gateway_name
- **add_route**: Whether to add a default route pointing to the internet gateway, assigned by referencing the input variable internet_gateway_add_route, with a default value of `true`
- **enable_ipv6**: Whether to enable IPv6 for the internet gateway, assigned by referencing the input variable internet_gateway_ipv6_enabled, with a default value of `false`

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name              = "tf_test_vpc"
subnet_name           = "tf_test_subnet"
internet_gateway_name = "tf_test_internet_gateway"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the VPC internet gateway
4. Run `terraform show` to view the created VPC internet gateway

## Reference Information

- [Huawei Cloud Elastic IP Product Documentation](https://support.huaweicloud.com/eip/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For EIP VPC Internet Gateway](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eip/vpc-with-internet-gateway)
