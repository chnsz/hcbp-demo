# Deploy Network

## Application Scenario

Cloud Container Instance (CCI) network is a network configuration function provided by the CCI service, used to provide network connectivity for container applications. By creating a CCI network, you can connect container applications to VPC networks, achieving interconnection between containers and other cloud resources. Automating CCI network creation through Terraform can ensure standardized and consistent network configuration, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create a CCI network, including the creation of VPC, subnet, security group, and namespace.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCI Namespace Resource (huaweicloud_cciv2_namespace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_namespace)
- [CCI Network Resource (huaweicloud_cciv2_network)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_network)

### Resource/Data Source Dependencies

```
huaweicloud_networking_secgroup
    └── huaweicloud_cciv2_network

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_cciv2_network

huaweicloud_cciv2_namespace
    └── huaweicloud_cciv2_network
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Security Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
  default     = "tf-test-secgroup"
}

# Create security group resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name, default value is "tf-test-secgroup"
- **delete_default_rules**: Whether to delete default rules, set to true

### 3. Create VPC Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
  default     = "tf-test-vpc"
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

# Create VPC resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name, default value is "tf-test-vpc"
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr, default value is "192.168.0.0/16"

### 4. Create VPC Subnet Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
  default     = "tf-test-subnet"
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "192.168.0.0/24"
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = "192.168.0.1"
}

# Create VPC subnet resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the previously created VPC resource (huaweicloud_vpc.test)
- **name**: The subnet name, assigned by referencing the input variable subnet_name, default value is "tf-test-subnet"
- **cidr**: The subnet CIDR block, assigned by referencing the input variable subnet_cidr, default value is "192.168.0.0/24"
- **gateway_ip**: The subnet gateway IP, assigned by referencing the input variable subnet_gateway_ip, default value is "192.168.0.1"

### 5. Create CCI Namespace Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CCI namespace resource:

```hcl
variable "namespace_name" {
  description = "The name of the CCI namespace"
  type        = string
}

# Create CCI namespace resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cciv2_namespace" "test" {
  name = var.namespace_name
}
```

**Parameter Description**:
- **name**: The namespace name, assigned by referencing the input variable namespace_name

### 6. Create CCI Network Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CCI network resource:

```hcl
variable "network_name" {
  description = "The name of the CCI network"
  type        = string
}

variable "warm_pool_size" {
  description = "The size of the warm pool for the network"
  type        = string
  default     = "10"
}

variable "warm_pool_recycle_interval" {
  description = "The recycle interval of the warm pool in hours"
  type        = string
  default     = "2"
}

# Create CCI network resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cciv2_network" "test" {
  depends_on = [huaweicloud_cciv2_namespace.test]

  namespace = huaweicloud_cciv2_namespace.test.name
  name      = var.network_name

  annotations = {
    "yangtse.io/project-id"                 = huaweicloud_cciv2_namespace.test.annotations["tenant.kubernetes.io/project-id"]
    "yangtse.io/domain-id"                  = huaweicloud_cciv2_namespace.test.annotations["tenant.kubernetes.io/domain-id"]
    "yangtse.io/warm-pool-size"             = var.warm_pool_size
    "yangtse.io/warm-pool-recycle-interval" = var.warm_pool_recycle_interval
  }

  subnets {
    subnet_id = huaweicloud_vpc_subnet.test.ipv4_subnet_id
  }

  security_group_ids = [huaweicloud_networking_secgroup.test.id]
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency relationship, ensuring the namespace resource is created before the network resource
- **namespace**: The namespace name, referencing the name of the previously created CCI namespace resource (huaweicloud_cciv2_namespace.test)
- **name**: The network name, assigned by referencing the input variable network_name
- **annotations.yangtse.io/project-id**: The project ID annotation, automatically inherited from the namespace annotations
- **annotations.yangtse.io/domain-id**: The domain ID annotation, automatically inherited from the namespace annotations
- **annotations.yangtse.io/warm-pool-size**: The warm pool size annotation, assigned by referencing the input variable warm_pool_size, default value is "10"
- **annotations.yangtse.io/warm-pool-recycle-interval**: The warm pool recycle interval annotation (in hours), assigned by referencing the input variable warm_pool_recycle_interval, default value is "2"
- **subnets.subnet_id**: The subnet ID, referencing the IPv4 subnet ID of the previously created VPC subnet resource (huaweicloud_vpc_subnet.test)
- **security_group_ids**: The security group ID list, referencing the ID of the previously created security group resource (huaweicloud_networking_secgroup.test)

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# CCI Network Configuration
network_name   = "tf-test-network"
namespace_name = "tf-test-namespace"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="network_name=test-network" -var="namespace_name=test-namespace"`
2. Environment variables: `export TF_VAR_network_name=test-network` and `export TF_VAR_namespace_name=test-namespace`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a CCI network:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CCI network
4. Run `terraform show` to view the details of the created CCI network

> Note: The network must be associated with at least one subnet. The warm pool configuration helps reduce pod startup time by pre-allocating resources. Network names must be unique within the namespace. The annotations yangtse.io/project-id and yangtse.io/domain-id are automatically inherited from the namespace.

## Reference Information

- [Huawei Cloud CCI Product Documentation](https://support.huaweicloud.com/cci/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Network](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cci/network)
