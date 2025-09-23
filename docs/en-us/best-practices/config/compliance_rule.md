# Deploy Compliance Rule

## Application Scenario

Configuration Audit (Config) is a one-stop compliance management service provided by Huawei Cloud, helping users continuously monitor and evaluate the configuration compliance of cloud resources. Config service provides pre-built compliance rule packages and custom rules, supporting multiple compliance frameworks and standards, helping enterprises establish a comprehensive compliance management system.

Compliance rules are a core function of Config service, used to define and execute specific compliance check policies. Through compliance rules, enterprises can monitor whether cloud resource configurations meet security and compliance requirements, and promptly discover and fix configuration risks. Compliance rules support multiple resource types and check conditions, including tag checks, configuration validation, security policies, etc., providing enterprises with comprehensive compliance management solutions. This best practice will introduce how to use Terraform to automatically deploy Config compliance rules, including VPC creation, ECS instance creation, compliance rule configuration, and rule evaluation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS Instance Resource (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [RMS Policy Assignment Resource (huaweicloud_rms_policy_assignment)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rms_policy_assignment)
- [RMS Policy Assignment Evaluation Resource (huaweicloud_rms_policy_assignment_evaluate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rms_policy_assignment_evaluate)

### Resource/Data Source Dependencies

```
huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_compute_instance.test
            └── huaweicloud_rms_policy_assignment.test
                └── huaweicloud_rms_policy_assignment_evaluate.test
    └── huaweicloud_networking_secgroup.test
        └── huaweicloud_compute_instance.test
            └── huaweicloud_rms_policy_assignment.test
                └── huaweicloud_rms_policy_assignment_evaluate.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create VPC Resource

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

### 3. Create VPC Subnet Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "Subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "Subnet CIDR block, if not specified, will calculate a subnet CIDR within the existing CIDR address block"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "Subnet gateway IP, if not specified, will calculate a gateway IP within the existing CIDR address block"
  type        = string
  default     = ""
  nullable    = false
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
- **cidr**: Subnet CIDR block, assigned by referencing the input variable subnet_cidr, automatically calculated if empty
- **gateway_ip**: Subnet gateway IP, assigned by referencing the input variable subnet_gateway_ip, automatically calculated if empty

### 4. Create Security Group Resource

Add the following script to the TF file to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "Security group name"
  type        = string
}

# Create a security group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to true

### 5. Create ECS Instance Resource

Add the following script to the TF file to instruct Terraform to create an ECS instance resource:

```hcl
variable "ecs_instance_name" {
  description = "ECS instance name"
  type        = string
}

variable "ecs_image_name" {
  description = "Image name for creating ECS instance"
  type        = string
  default     = "Ubuntu 20.04 server 64bit"
}

variable "ecs_flavor_name" {
  description = "ECS instance flavor name"
  type        = string
  default     = "s6.small.1"
}

variable "availability_zone" {
  description = "Availability zone where the ECS instance is located"
  type        = string
}

variable "ecs_tags" {
  description = "ECS instance tags"
  type        = map(string)
  default = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# Create an ECS instance resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  image_name         = var.ecs_image_name
  flavor_name        = var.ecs_flavor_name
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  availability_zone  = var.availability_zone

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  tags = var.ecs_tags
}
```

**Parameter Description**:
- **name**: Instance name, assigned by referencing the input variable ecs_instance_name
- **image_name**: Image name, assigned by referencing the input variable ecs_image_name
- **flavor_name**: Flavor name, assigned by referencing the input variable ecs_flavor_name
- **security_group_ids**: Security group ID list, assigned by referencing the security group resource (huaweicloud_networking_secgroup.test) ID
- **availability_zone**: Availability zone, assigned by referencing the input variable availability_zone
- **network**: Network configuration, assigned by referencing the VPC subnet resource (huaweicloud_vpc_subnet.test) ID
- **tags**: Tags, assigned by referencing the input variable ecs_tags

### 6. Create Compliance Rule

Add the following script to the TF file to instruct Terraform to create a compliance rule resource:

```hcl
variable "policy_assignment_name" {
  description = "Policy assignment name"
  type        = string
}

variable "policy_assignment_description" {
  description = "Policy assignment description"
  type        = string
  default     = "Check if ECS instances have required tags"
}

variable "policy_definition_id" {
  description = "Policy definition ID"
  type        = string
}

variable "policy_assignment_policy_filter" {
  description = "Configuration for filtering resources"

  type = list(object({
    region            = string
    resource_provider = string
    resource_type     = string
    resource_id       = string
    tag_key           = string
    tag_value         = string
  }))

  default = []
}

variable "policy_assignment_parameters" {
  description = "Policy assignment parameters"
  type        = map(string)
  default = {
    "tagKeys" = "[\"Owner\",\"Env\"]"
  }
}

variable "policy_assignment_tags" {
  description = "Policy assignment tags"
  type        = map(string)
  default = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# Create a compliance rule resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_rms_policy_assignment" "test" {
  name                 = var.policy_assignment_name
  description          = var.policy_assignment_description
  policy_definition_id = var.policy_definition_id

  dynamic "policy_filter" {
    for_each = var.policy_assignment_policy_filter

    content {
      region            = policy_filter.value.region
      resource_provider = policy_filter.value.resource_provider
      resource_type     = policy_filter.value.resource_type
      resource_id       = policy_filter.value.resource_id
      tag_key           = policy_filter.value.tag_key
      tag_value         = policy_filter.value.tag_value
    }
  }

  parameters = var.policy_assignment_parameters

  tags = var.policy_assignment_tags
}
```

**Parameter Description**:
- **name**: Policy assignment name, assigned by referencing the input variable policy_assignment_name
- **description**: Policy assignment description, assigned by referencing the input variable policy_assignment_description
- **policy_definition_id**: Policy definition ID, assigned by referencing the input variable policy_definition_id
- **policy_filter**: Policy filter, dynamically creates resource filtering conditions
  - **region**: Region, assigned by referencing the region in the filter configuration
  - **resource_provider**: Resource provider, assigned by referencing the resource_provider in the filter configuration
  - **resource_type**: Resource type, assigned by referencing the resource_type in the filter configuration
  - **resource_id**: Resource ID, assigned by referencing the resource_id in the filter configuration
  - **tag_key**: Tag key, assigned by referencing the tag_key in the filter configuration
  - **tag_value**: Tag value, assigned by referencing the tag_value in the filter configuration
- **parameters**: Parameters, assigned by referencing the input variable policy_assignment_parameters
- **tags**: Tags, assigned by referencing the input variable policy_assignment_tags

### 7. Evaluate Compliance Rule

Add the following script to the TF file to instruct Terraform to evaluate the compliance rule:

```hcl
# Evaluate the compliance rule under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_rms_policy_assignment_evaluate" "test" {
  policy_assignment_id = huaweicloud_rms_policy_assignment.test.id
}
```

**Parameter Description**:
- **policy_assignment_id**: Policy assignment ID, assigned by referencing the compliance rule resource (huaweicloud_rms_policy_assignment.test) ID

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Network configuration
vpc_name               = "tf_test_vpc"
vpc_cidr               = "192.168.0.0/16"
subnet_name            = "tf_test_subnet"
subnet_cidr            = "192.168.0.0/24"
subnet_gateway_ip      = "192.168.0.1"
security_group_name    = "tf_test_security_group"

# ECS instance configuration
ecs_instance_name      = "tf_test_ecs"
ecs_image_name         = "Ubuntu 20.04 server 64bit"
ecs_flavor_name        = "s6.small.1"
availability_zone      = "cn-north-4a"
ecs_tags               = {
  "Owner" = "terraform"
  "Env"   = "test"
}

# Compliance rule configuration
policy_assignment_name = "tf_test_policy_assignment_name"
policy_assignment_description = "Check if ECS instances have required tags"
policy_definition_id   = "tf_test_policy_definition_id"
policy_assignment_policy_filter = [
  {
    region            = "cn-north-4"
    resource_provider = "hws.resource.type.ecs"
    resource_type     = "hws.resource.type.ecs"
    resource_id       = ""
    tag_key           = "Owner"
    tag_value         = "terraform"
  }
]
policy_assignment_parameters = {
  "tagKeys" = "[\"Owner\",\"Env\"]"
}
policy_assignment_tags = {
  "Owner" = "terraform"
  "Env"   = "test"
}
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="policy_assignment_name=my-policy"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the compliance rule
4. Run `terraform show` to view the details of the created compliance rule

## Reference Information

- [Huawei Cloud Config Product Documentation](https://support.huaweicloud.com/rms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Config Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rms)
