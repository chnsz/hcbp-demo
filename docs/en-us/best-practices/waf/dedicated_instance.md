# Deploy Professional Edition Instance

## Application Scenario

Huawei Cloud Web Application Firewall (WAF) Professional Edition is a Web security protection service based on dedicated resources that can effectively defend against various common Web attacks such as SQL injection, XSS cross-site scripting, web trojan upload, command injection, and more.
WAF Professional Edition instances provide dedicated resources, offering more customized Web security protection for enterprises, suitable for scenarios with high-performance requirements for Web application security protection and strict requirements for compliance and data isolation.
This best practice will introduce how to use Terraform to automatically deploy a WAF Professional Edition instance.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zone List Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Compute Flavor List Query Data Source (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [WAF Professional Edition Instance Resource (huaweicloud_waf_dedicated_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_dedicated_instance)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── huaweicloud_vpc_subnet
    ├── data.huaweicloud_compute_flavors
    └── huaweicloud_waf_dedicated_instance

data.huaweicloud_compute_flavors
    └── huaweicloud_waf_dedicated_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_waf_dedicated_instance

huaweicloud_networking_secgroup
    └── huaweicloud_waf_dedicated_instance
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
  default     = "192.168.0.0/16"
}

# Create a VPC resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to deploy WAF Professional Edition instances
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable vpc_cidr, default value is "192.168.0.0/16"

### 3. Query Availability Zones Required for WAF Instance Resource Creation Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the results of which are used to create WAF Professional Edition instances:

```hcl
variable "availability_zone" {
  description = "The availability zone to which the dedicated instance belongs"
  type        = string
  default     = ""
}

# Get all availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create WAF Professional Edition instances
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: Creation count of the data source, used to control whether to execute the availability zone list query data source, only creates the data source when `var.availability_zone` is empty (i.e., executes availability zone list query)

### 4. Create VPC Subnet Resource

Add the following script to the TF file to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
}

# Create a VPC subnet resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to deploy WAF Professional Edition instances
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  name              = var.subnet_name
  cidr              = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip        = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
}
```

**Parameter Description**:
- **vpc_id**: VPC ID that the subnet belongs to, referencing the ID of the previously created VPC resource
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block, if subnet_cidr is empty, uses cidrsubnet function to divide a subnet segment from the VPC's CIDR block, otherwise uses the specified subnet_cidr
- **gateway_ip**: Subnet gateway IP, if subnet_gateway_ip is empty, uses cidrhost function to get the first IP address from the subnet segment as gateway IP, otherwise uses the specified subnet_gateway_ip
- **availability_zone**: Availability zone where the subnet is located, if availability_zone is empty, uses the first availability zone from the availability zone list query data source, otherwise uses the specified availability_zone

### 5. Query Compute Flavors Required for WAF Instance Resource Creation Through Data Source

Add the following script to the TF file to instruct Terraform to query compute flavors that meet the conditions:

```hcl
variable "dedicated_instance_flavor_id" {
  description = "The flavor ID of the dedicated instance"
  type        = string
  default     = ""
}

variable "dedicated_instance_performance_type" {
  description = "The performance type of the dedicated instance"
  type        = string
  default     = "normal"
}

variable "dedicated_instance_cpu_core_count" {
  description = "The number of the dedicated instance CPU cores"
  type        = number
  default     = 4
}

variable "dedicated_instance_memory_size" {
  description = "The memory size of the dedicated instance"
  type        = number
  default     = 8
}

# Get all compute flavor information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block) that meets specific conditions, used to create WAF Professional Edition instances
data "huaweicloud_compute_flavors" "test" {
  count = var.dedicated_instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  performance_type  = var.dedicated_instance_performance_type
  cpu_core_count    = var.dedicated_instance_cpu_core_count
  memory_size       = var.dedicated_instance_memory_size
}
```

**Parameter Description**:
- **count**: Creation count of the data source, used to control whether to execute the compute flavor list query data source, only creates the data source when `var.dedicated_instance_flavor_id` is empty
- **availability_zone**: Availability zone where the compute flavor is located, if availability_zone is empty, uses the first availability zone from the availability zone list query data source, otherwise uses the specified availability_zone
- **performance_type**: Performance type, assigned by referencing the input variable dedicated_instance_performance_type, default value is "normal"
- **cpu_core_count**: CPU core count, assigned by referencing the input variable dedicated_instance_cpu_core_count, default value is 4
- **memory_size**: Memory size (GB), assigned by referencing the input variable dedicated_instance_memory_size, default value is 8

### 6. Create Security Group Resource

Add the following script to the TF file to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

# Create a security group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to deploy WAF Professional Edition instances
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to true to delete default rules

### 7. Create WAF Professional Edition Instance Resource

Add the following script to the TF file to instruct Terraform to create a WAF Professional Edition instance resource:

```hcl
variable "dedicated_instance_name" {
  description = "The WAF dedicated instance name"
  type        = string
}

variable "dedicated_instance_specification_code" {
  description = "The specification code of the dedicated instance"
  type        = string
  default     = "waf.instance.professional"
}

# Create a WAF Professional Edition instance resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_waf_dedicated_instance" "test" {
  name               = var.dedicated_instance_name
  specification_code = var.dedicated_instance_specification_code
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  available_zone     = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  ecs_flavor         = var.dedicated_instance_flavor_id == "" ? data.huaweicloud_compute_flavors.test[0].ids[0] : var.dedicated_instance_flavor_id

  security_group = [
    huaweicloud_networking_secgroup.test.id
  ]
}
```

**Parameter Description**:
- **name**: WAF Professional Edition instance name, assigned by referencing the input variable dedicated_instance_name
- **specification_code**: WAF Professional Edition instance specification code, assigned by referencing the input variable dedicated_instance_specification_code, default value is "waf.instance.professional"
- **vpc_id**: VPC ID where the WAF Professional Edition instance is located, referencing the ID of the previously created VPC resource
- **subnet_id**: Subnet ID where the WAF Professional Edition instance is located, referencing the ID of the previously created subnet resource
- **available_zone**: Availability zone where the WAF Professional Edition instance is located, if availability_zone is empty, uses the first availability zone from the availability zone list query data source, otherwise uses the specified availability_zone
- **ecs_flavor**: Compute flavor used by the WAF Professional Edition instance, if dedicated_instance_flavor_id is empty, uses the first flavor from the compute flavor list query data source, otherwise uses the specified dedicated_instance_flavor_id
- **security_group**: Security group ID list used by the WAF Professional Edition instance, referencing the ID of the previously created security group resource

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC configuration
vpc_name = "tf_test_vpc"
vpc_cidr = "192.168.0.0/16"

# Subnet configuration
subnet_name = "tf_test_subnet"

# WAF Professional Edition instance configuration
dedicated_instance_name               = "tf_test_waf_dedicated_instance"
dedicated_instance_specification_code = "waf.instance.professional"
dedicated_instance_performance_type   = "normal"
dedicated_instance_cpu_core_count     = 4
dedicated_instance_memory_size        = 8

# Security group configuration
security_group_name = "tf_test_secgroup"
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

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating WAF Professional Edition instances
4. Run `terraform show` to view the created WAF Professional Edition instance details

## Reference Information

- [Huawei Cloud WAF Product Documentation](https://support.huaweicloud.com/waf/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [WAF Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/waf)
