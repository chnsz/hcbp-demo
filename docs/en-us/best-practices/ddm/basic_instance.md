# Deploy Basic Instance

## Application Scenario

Distributed Database Middleware (DDM) is a MySQL-compatible distributed relational database middleware that focuses on solving database distributed scaling issues, breaking through capacity and performance bottlenecks of traditional databases to enable highly concurrent access to massive volumes of data. By creating a basic DDM instance, you can quickly set up a distributed database middleware environment with network isolation, secure access control, and flexible billing, providing a foundation for subsequent capabilities such as database and table sharding and read/write splitting. This best practice introduces how to use Terraform to automatically deploy a basic DDM instance, including creating VPC, subnet, and security group, as well as configuring engine, flavor, node number, billing mode, and other parameters.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DDM Engines Data Source (data.huaweicloud_ddm_engines)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ddm_engines)
- [DDM Flavors Data Source (data.huaweicloud_ddm_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ddm_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DDM Instance Resource (huaweicloud_ddm_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ddm_instance)

### Resource/Data Source Dependencies

```text
data.huaweicloud_availability_zones
    └── huaweicloud_ddm_instance

data.huaweicloud_ddm_engines
    ├── data.huaweicloud_ddm_flavors
    │   └── huaweicloud_ddm_instance
    └── huaweicloud_ddm_instance

data.huaweicloud_ddm_flavors
    └── huaweicloud_ddm_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_ddm_instance

huaweicloud_networking_secgroup
    └── huaweicloud_ddm_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the configuration introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the DDM instance:

```hcl
variable "availability_zones" {
  description = "The availability zones to which the DDM instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

# Query all availability zones in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the DDM instance
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the availability zones data source. The data source is created only when `var.availability_zones` is an empty list.

### 3. Create VPC Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

# Create a VPC resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the DDM instance
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable `vpc_name`
- **cidr**: CIDR block of the VPC, assigned by referencing the input variable `vpc_cidr`

### 4. Create VPC Subnet Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC subnet resource:

```hcl
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
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

# Create a VPC subnet resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the DDM instance
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: ID of the VPC that the subnet belongs to, assigned by referencing the ID of the VPC resource
- **name**: Subnet name, assigned by referencing the input variable `subnet_name`
- **cidr**: CIDR block of the subnet. If the input variable is empty, it is calculated automatically based on the VPC CIDR; otherwise, the input variable value is used
- **gateway_ip**: Gateway IP address of the subnet. If the input variable is empty, it is calculated automatically; otherwise, the input variable value is used

### 5. Create Security Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# Create a security group resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to deploy the DDM instance
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable `security_group_name`
- **delete_default_rules**: Whether to delete default security group rules. Fixed to `true` in this practice

### 6. Query DDM Engines

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the DDM instance and query flavors:

```hcl
variable "instance_engine_id" {
  description = "The engine ID of the DDM instance"
  type        = string
  default     = ""
  nullable    = false
}

# Query all DDM engines in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the DDM instance
data "huaweicloud_ddm_engines" "test" {
  count = var.instance_engine_id == "" ? 1 : 0
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the DDM engines data source. The data source is created only when `var.instance_engine_id` is empty.

### 7. Query DDM Flavors

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query. The query result is used to create the DDM instance:

```hcl
variable "instance_flavor_id" {
  description = "The flavor ID of the DDM instance"
  type        = string
  default     = ""
  nullable    = false
}

# Query DDM flavors of the specified engine in the specified region (defaults to the region specified in the provider block when the region parameter is omitted), used to create the DDM instance
data "huaweicloud_ddm_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  engine_id = var.instance_engine_id == "" ? try(data.huaweicloud_ddm_engines.test[0].engines[0].id, null) : var.instance_engine_id
}
```

**Parameter Description**:
- **count**: Number of data source instances, used to control whether to execute the DDM flavors data source. The data source is created only when `var.instance_flavor_id` is empty.
- **engine_id**: DDM engine ID. If the input variable is empty, it is assigned based on the DDM engines data source result; otherwise, the input variable value is used

### 8. Create DDM Instance

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DDM instance resource:

```hcl
variable "instance_name" {
  description = "The name of the DDM instance"
  type        = string
}

variable "instance_node_num" {
  description = "The number of nodes in the DDM instance"
  type        = number
  default     = 2
}

variable "instance_admin_user_name" {
  description = "The administrator username of the DDM instance"
  type        = string
  default     = ""
}

variable "instance_admin_user_password" {
  description = "The administrator password of the DDM instance"
  sensitive   = true
  type        = string
  default     = ""
}

variable "instance_parameters" {
  description = "The parameters of the DDM instance"

  type = list(object({
    name  = string
    value = string
  }))

  default = []
}

variable "charging_mode" {
  description = "The charging mode of the DDM instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the DDM instance"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the DDM instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the DDM instance"
  type        = string
  default     = "false"
}

# Create a DDM instance resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_ddm_instance" "test" {
  name               = var.instance_name
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  engine_id          = var.instance_engine_id == "" ? try(data.huaweicloud_ddm_engines.test[0].engines[0].id, null) : var.instance_engine_id
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_ddm_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  node_num           = var.instance_node_num
  admin_user         = var.instance_admin_user_name
  admin_password     = var.instance_admin_user_password

  dynamic "parameters" {
    for_each = var.instance_parameters

    content {
      name  = parameters.value.name
      value = parameters.value.value
    }
  }

  charging_mode = var.charging_mode
  period_unit   = var.period_unit
  period        = var.period
  auto_renew    = var.auto_renew
}
```

**Parameter Description**:
- **name**: DDM instance name, assigned by referencing the input variable `instance_name`
- **availability_zones**: Availability zone list of the DDM instance. If the input variable is an empty list, it is assigned based on the availability zones data source result; otherwise, the input variable value is used
- **engine_id**: Engine ID of the DDM instance. If the input variable is empty, it is assigned based on the DDM engines data source result; otherwise, the input variable value is used
- **flavor_id**: Flavor ID of the DDM instance. If the input variable is empty, it is assigned based on the DDM flavors data source result; otherwise, the input variable value is used
- **vpc_id**: ID of the VPC that the DDM instance belongs to, assigned by referencing the ID of the VPC resource
- **subnet_id**: ID of the subnet that the DDM instance belongs to, assigned by referencing the ID of the VPC subnet resource
- **security_group_id**: ID of the security group that the DDM instance belongs to, assigned by referencing the ID of the security group resource
- **node_num**: Number of nodes in the DDM instance, assigned by referencing the input variable `instance_node_num`
- **admin_user**: Administrator username of the DDM instance, assigned by referencing the input variable `instance_admin_user_name`
- **admin_password**: Administrator password of the DDM instance, assigned by referencing the input variable `instance_admin_user_password`
- **parameters**: Parameter configuration of the DDM instance, created by the dynamic block `dynamic "parameters"` based on the input variable `instance_parameters`
  - **name**: Parameter name, assigned by referencing `name` in the input variable
  - **value**: Parameter value, assigned by referencing `value` in the input variable
- **charging_mode**: Charging mode of the DDM instance, assigned by referencing the input variable `charging_mode`
- **period_unit**: Period unit of the DDM instance, assigned by referencing the input variable `period_unit`
- **period**: Period of the DDM instance, assigned by referencing the input variable `period`
- **auto_renew**: Whether to enable auto renew, assigned by referencing the input variable `auto_renew`

> Note: Keep sensitive information such as the administrator password secure and never commit it to version control.

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables for configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory. Example content is as follows:

```hcl
# Network and security group configuration
vpc_name            = "tf_test_instance"
subnet_name         = "tf_test_instance"
security_group_name = "tf_test_instance"

# DDM instance configuration
instance_name = "tf_test_instance"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; for other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="vpc_name=tf_test_instance" -var="instance_name=tf_test_instance"`
2. Environment variables: `export TF_VAR_instance_name=tf_test_instance`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DDM instance
4. Run `terraform show` to view the created DDM instance

## Reference Information

- [Huawei Cloud DDM Product Documentation](https://support.huaweicloud.com/ddm/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DDM Basic Instance Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ddm/basic-instance)
