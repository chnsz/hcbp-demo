# Deploy Account

## Application Scenario

Distributed Database Middleware (DDM) is a MySQL-compatible distributed relational database middleware that focuses on solving database distributed scaling issues, breaking through capacity and performance bottlenecks of traditional databases to enable highly concurrent access to massive volumes of data. DDM accounts control access to logical databases and can be assigned basic permissions based on business needs. This best practice introduces how to use Terraform to automatically deploy a DDM instance and account, including creating VPC, subnet, and security group, configuring engine, flavor, and node number, and creating a DDM account with specified permissions.

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
- [Random Password Resource (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [DDM Account Resource (huaweicloud_ddm_account)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ddm_account)

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

huaweicloud_ddm_instance
    └── huaweicloud_ddm_account

random_password
    └── huaweicloud_ddm_account
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

variable "instance_parameters" {
  description = "The parameters of the DDM instance"

  type = list(object({
    name  = string
    value = string
  }))

  default = []
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

  dynamic "parameters" {
    for_each = var.instance_parameters

    content {
      name  = parameters.value.name
      value = parameters.value.value
    }
  }
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
- **parameters**: Parameter configuration of the DDM instance, created by the dynamic block `dynamic "parameters"` based on the input variable `instance_parameters`
  - **name**: Parameter name, assigned by referencing `name` in the input variable
  - **value**: Parameter value, assigned by referencing `value` in the input variable

### 9. Create Random Password

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a random password resource:

```hcl
variable "account_password" {
  description = "The password of the DDM account"
  sensitive   = true
  type        = string
  default     = ""
  nullable    = false
}

# Create a random password resource, used to generate the DDM account password when the account password is not specified
resource "random_password" "test" {
  count = var.account_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "!@#%^*-_+?"
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**Parameter Description**:
- **count**: Number of resource instances, used to control whether to create the random password resource. The resource is created only when `var.account_password` is empty
- **length**: Password length. Fixed to `12` in this practice
- **special**: Whether to include special characters. Fixed to `true` in this practice
- **override_special**: Allowed special character set. Fixed to `"!@#%^*-_+?"` in this practice
- **min_upper**: Minimum number of uppercase letters in the password. Fixed to `1` in this practice
- **min_lower**: Minimum number of lowercase letters in the password. Fixed to `1` in this practice
- **min_numeric**: Minimum number of numeric characters in the password. Fixed to `1` in this practice
- **min_special**: Minimum number of special characters in the password. Fixed to `1` in this practice

### 10. Create DDM Account

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DDM account resource:

```hcl
variable "account_name" {
  description = "The name of the DDM account"
  type        = string
}

variable "account_permissions" {
  description = "The basic permissions of the DDM account"
  type        = list(string)
  default     = ["SELECT"]
  nullable    = false
}

variable "account_description" {
  description = "The description of the DDM account"
  type        = string
  default     = ""
}

# Create a DDM account resource in the specified region (defaults to the region specified in the provider block when the region parameter is omitted)
resource "huaweicloud_ddm_account" "test" {
  instance_id = huaweicloud_ddm_instance.test.id
  name        = var.account_name
  password    = var.account_password == "" ? try(random_password.test[0].result, null) : var.account_password
  permissions = var.account_permissions
  description = var.account_description
}
```

**Parameter Description**:
- **instance_id**: ID of the DDM instance that the account belongs to, assigned by referencing the ID of the DDM instance resource
- **name**: DDM account name, assigned by referencing the input variable `account_name`
- **password**: DDM account password. If the input variable is empty, it is assigned based on the random password resource result; otherwise, the input variable value is used
- **permissions**: Basic permission list of the DDM account, assigned by referencing the input variable `account_permissions`
- **description**: Description of the DDM account, assigned by referencing the input variable `account_description`

> Note: Keep sensitive information such as the account password secure and never commit it to version control.

### 11. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables for configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through a `tfvars` file, which can avoid repeated input each time it is executed.

Create a `terraform.tfvars` file in the working directory. Example content is as follows:

```hcl
# Network and security group configuration
vpc_name            = "tf_test_account"
subnet_name         = "tf_test_account"
security_group_name = "tf_test_account"

# DDM instance configuration
instance_name = "tf_test_account"

# DDM account configuration
account_name = "tf_test_account"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows Terraform to automatically import the content of the `tfvars` file when executing Terraform commands; for other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command-line parameters: `terraform apply -var="vpc_name=tf_test_account" -var="account_name=tf_test_account"`
2. Environment variables: `export TF_VAR_account_name=tf_test_account`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command-line parameters > variable files > environment variables > default values.

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, perform the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DDM account
4. Run `terraform show` to view the created DDM account

## Reference Information

- [Huawei Cloud DDM Product Documentation](https://support.huaweicloud.com/ddm/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DDM Account Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ddm/ddm-account)
