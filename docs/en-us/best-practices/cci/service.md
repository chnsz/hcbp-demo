# Deploy Service

## Application Scenario

Cloud Container Instance (CCI) service is a service discovery and load balancing function provided by the CCI service, used to provide a unified access entry for container applications. By creating a CCI service, you can expose container applications to external access, achieving service load balancing and traffic distribution. Automating CCI service creation through Terraform can ensure standardized and consistent service configuration, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create a CCI service, including the creation of VPC, subnet, security group, namespace, and ELB load balancer.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCI Namespace Resource (huaweicloud_cciv2_namespace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_namespace)
- [ELB Load Balancer Resource (huaweicloud_elb_loadbalancer)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/elb_loadbalancer)
- [CCI Service Resource (huaweicloud_cciv2_service)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cciv2_service)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_elb_loadbalancer

huaweicloud_networking_secgroup
    └── huaweicloud_cciv2_service

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_elb_loadbalancer
        └── huaweicloud_cciv2_service

huaweicloud_cciv2_namespace
    ├── huaweicloud_elb_loadbalancer
    └── huaweicloud_cciv2_service

huaweicloud_elb_loadbalancer
    └── huaweicloud_cciv2_service
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zone Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the query results of which are used to create an ELB load balancer:

```hcl
# Get all availability zone information in the specified region (defaults to the region specified in the provider block when region parameter is omitted), used to create ELB load balancer
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
This data source does not require additional parameters and queries all available availability zone information in the current region by default.

### 3. Create Security Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "The name of security group"
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

### 4. Create VPC Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  description = "The name of VPC"
  type        = string
  default     = "tf-test-vpc"
}

variable "vpc_cidr" {
  description = "The CIDR block of VPC"
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

### 5. Create VPC Subnet Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "The name of subnet"
  type        = string
  default     = "tf-test-subnet"
}

variable "subnet_cidr" {
  description = "The CIDR block of subnet"
  type        = string
  default     = "192.168.0.0/24"
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of subnet"
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

### 6. Create CCI Namespace Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CCI namespace resource:

```hcl
variable "namespace_name" {
  description = "The name of CCI namespace"
  type        = string
}

# Create CCI namespace resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cciv2_namespace" "test" {
  name = var.namespace_name
}
```

**Parameter Description**:
- **name**: The namespace name, assigned by referencing the input variable namespace_name

### 7. Create ELB Load Balancer Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an ELB load balancer resource:

```hcl
variable "elb_name" {
  description = "The name of ELB load balancer"
  type        = string
}

# Create ELB load balancer resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_elb_loadbalancer" "test" {
  depends_on = [huaweicloud_cciv2_namespace.test]

  name              = var.elb_name
  cross_vpc_backend = true
  vpc_id            = huaweicloud_vpc.test.id
  ipv4_subnet_id    = huaweicloud_vpc_subnet.test.ipv4_subnet_id

  availability_zone = [
    try(data.huaweicloud_availability_zones.test.names[1], "")
  ]
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency relationship, ensuring the namespace resource is created before the ELB load balancer resource
- **name**: The load balancer name, assigned by referencing the input variable elb_name
- **cross_vpc_backend**: Whether to support cross-VPC backend, set to true
- **vpc_id**: The VPC ID, referencing the ID of the previously created VPC resource (huaweicloud_vpc.test)
- **ipv4_subnet_id**: The IPv4 subnet ID, referencing the IPv4 subnet ID of the previously created VPC subnet resource (huaweicloud_vpc_subnet.test)
- **availability_zone**: The availability zone list, assigned based on the query results of the availability zone list data source (data.huaweicloud_availability_zones), using the second availability zone

### 8. Create CCI Service Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CCI service resource:

```hcl
variable "service_name" {
  description = "The name of CCI service"
  type        = string
}

variable "selector_app" {
  description = "The app label of selector"
  type        = string
  default     = "test1"
}

variable "service_type" {
  description = "The type of service"
  type        = string
  default     = "LoadBalancer"
}

# Create CCI service resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cciv2_service" "test" {
  depends_on = [huaweicloud_elb_loadbalancer.test]

  namespace = huaweicloud_cciv2_namespace.test.name
  name      = var.service_name

  annotations = {
    "kubernetes.io/elb.class" = "elb"
    "kubernetes.io/elb.id"    = huaweicloud_elb_loadbalancer.test.id
  }

  ports {
    name         = "test"
    app_protocol = "TCP"
    protocol     = "TCP"
    port         = 87
    target_port  = 65529
  }

  selector = {
    app = var.selector_app
  }

  type = var.service_type

  lifecycle {
    ignore_changes = [
      annotations,
    ]
  }
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency relationship, ensuring the ELB load balancer resource is created before the CCI service resource
- **namespace**: The namespace name, referencing the name of the previously created CCI namespace resource (huaweicloud_cciv2_namespace.test)
- **name**: The service name, assigned by referencing the input variable service_name
- **annotations.kubernetes.io/elb.class**: The ELB type annotation, set to "elb"
- **annotations.kubernetes.io/elb.id**: The ELB ID annotation, referencing the ID of the previously created ELB load balancer resource (huaweicloud_elb_loadbalancer.test)
- **ports.name**: The port name, set to "test"
- **ports.app_protocol**: The application protocol, set to "TCP"
- **ports.protocol**: The protocol type, set to "TCP"
- **ports.port**: The service port, set to 87
- **ports.target_port**: The target port, set to 65529
- **selector.app**: The selector application label, assigned by referencing the input variable selector_app, default value is "test1"
- **type**: The service type, assigned by referencing the input variable service_type, default value is "LoadBalancer"

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# CCI Service Configuration
elb_name       = "tf-test-elb"
service_name   = "tf-test-service"
namespace_name = "tf-test-namespace"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="service_name=test-service" -var="namespace_name=test-namespace"`
2. Environment variables: `export TF_VAR_service_name=test-service` and `export TF_VAR_namespace_name=test-namespace`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a CCI service:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CCI service
4. Run `terraform show` to view the details of the created CCI service

> Note: The service must be created in an existing namespace. The selector label must match the Pod label. The ELB load balancer must be created in advance.

## Reference Information

- [Huawei Cloud CCI Product Documentation](https://support.huaweicloud.com/cci/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Service](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cci/service)

