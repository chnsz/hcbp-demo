# Language Introduction

The Terraform Configuration Language (abbreviated as HCL) is an important credential for describing and operating infrastructure managed through Terraform.

Through configuration files written in this language, you can tell Terraform:
- Which plugins to install
- Which infrastructure to create
- Which data to fetch

The Terraform Configuration Language also allows you to define dependencies between resources and manage multiple similar resources using a single configuration block. It features simple configuration, strong readability, and compatibility with JSON syntax.

## About the Terraform Language

The main purpose of the Terraform language is to declare **resources**, which represent infrastructure objects. All other language features exist only to make resource definitions more flexible and convenient.

A **Terraform configuration** is a complete document in the Terraform language that tells Terraform how to manage a given collection of infrastructure. A configuration can consist of multiple files and directories.

### Basic Syntax Elements

The syntax of the Terraform language consists of the following basic elements:

```hcl
<BLOCK TYPE> "<BLOCK LABEL>" "<BLOCK LABEL>" {
  # Block content (including but not limited to meta-arguments, variable definitions)
  <IDENTIFIER> = <EXPRESSION>
}
```

1. **Blocks**: Containers for other content, usually representing the configuration of some kind of object, such as resources. Blocks have a **block type**, can have zero or more **labels**, and have a **block body** that contains any number of arguments and nested blocks. Most of Terraform's features are controlled by top-level blocks in configuration files.

2. **Arguments**: Assign values to names. They appear within blocks.

3. **Expressions**: Represent a value, either literally or by referencing and combining other values. They appear as values for arguments or within other expressions.

### Declarative Nature

The Terraform language is declarative, describing the intended goal rather than the steps to reach that goal. The ordering of blocks and files is generally not significant; they will be executed by Terraform drawing an execution graph and determining the execution order when the Terraform configuration file is executed. When declaring, you only need to consider the implicit and explicit reference relationships between resources.

## Configuration Example

The following example describes a simple network topology for Huawei Cloud, showcasing the overall structure and syntax of the Terraform language. Similar configurations can be applied to resource types provided by other services. Actual network configurations typically contain additional elements not shown here.

```hcl
terraform {
  required_providers {
    huaweicloud = {
      source  = "huaweicloud/huaweicloud"
      version = ">= 1.76.0"
    }
  }
}

variable "region_name" {}

provider "huaweicloud" {
  region = var.region_name
}

variable "vpc_name" {
  description = "Name of the VPC, required parameter"
}

variable "vpc_cidr_block" {
  description = "/16 CIDR range definition for the VPC, such as 192.168.0.0/16, optional parameter"
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  # Create VPC network by referencing the final values of vpc_name and vpc_cidr_block variables (optional parameters use default values if omitted)
  name = var.vpc_name
  cidr = var.vpc_cidr_block
}

variable "subnet_name" {
  description = "Name of the subnet, required parameter"
}

variable "availability_zones" {
  description = "Availability zone information for subnets to be created in the VPC"
  type        = list(string)
}

resource "huaweicloud_vpc_subnet" "test" {
  # Create a subnet for each given availability zone by iterating through the availability zone list
  count = length(var.availability_zones)

  # For each subnet, use one of the specified availability zones
  availability_zone = var.availability_zones[count.index]

  # Build implicit dependencies by referencing properties of the huaweicloud_vpc.test object (required parameters do not constitute implicit dependencies since their values are determined before creation), letting Terraform know that subnets must wait for VPC creation to complete before starting creation
  vpc_id = huaweicloud_vpc.test.id

  name = var.subnet_name
  # Built-in functions and operators can be used for simple transformations of values, such as calculating subnet addresses
  # Create a /20 prefix and continuous address block for each subnet
  cidr = cidrsubnet(huaweicloud_vpc.test.cidr, 4, count.index+1)
  # Functions can reference calculation results of other functions
  # Use the first IP address in the address range (excluding broadcast address) as the gateway for the subnet
  gateway_ip = cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 4, count.index+1), 1)
}
```

## Basic Learning

To master the core concepts of the Terraform configuration language, it is recommended to learn the basics in the following order:

1. [Configuration Language Basics](https://developer.hashicorp.com/terraform/language)
   - **Files and Directory Structure**: Understand how Terraform configuration files are organized, including `.tf` files, variable files, state files, etc.
   - **Syntax Rules**: Master the basic rules of HCL syntax, including correct writing of blocks, arguments, and expressions
   - **Resources and Data Sources**: Understand the difference between resources and data sources, and their roles in configuration

2. [Resources and Data Sources](https://developer.hashicorp.com/terraform/language/resources)
   - **Resource Behavior**: Understand resource lifecycle behaviors such as state management, creation, updates, and deletion
   - **Dependencies**: Master implicit and explicit dependencies between resources, and how Terraform resolves these dependencies
   - **Meta-arguments**: Learn how to use meta-arguments like count, for_each, depends_on

3. [Variables and Outputs](https://developer.hashicorp.com/terraform/language/values)
   - **Variable Definition**: Master input variable definitions, type constraints, default value settings, etc.
   - **Output Values**: Understand the definition and use of output values, and how to reference them in other configurations
   - **Local Values**: Learn the concept and use cases of local values to improve configuration maintainability

4. [Expressions](https://developer.hashicorp.com/terraform/language/expressions)
   - **Reference Expressions**: Master how to reference resource attributes, variables, data sources, etc.
   - **Operators**: Learn the use of arithmetic, logical, and comparison operators
   - **Conditional Expressions**: Understand the syntax and application scenarios of conditional expressions

## Advanced Learning

After mastering the basics, you can further learn the following advanced features:

1. [Functions](https://developer.hashicorp.com/terraform/language/functions)
   - Built-in functions
   - Numeric functions
   - String functions
   - Collection functions

2. [Modules](https://developer.hashicorp.com/terraform/language/modules)
   - Module usage
   - Module creation
   - Module composition

> **Practice Recommendation**: Try the [Terraform Configuration Writing Tutorial](https://developer.hashicorp.com/terraform/tutorials/configuration-language).

Learn more through the following links:
- [Terraform Official Documentation](https://developer.hashicorp.com/terraform/language)
