# Terraform Configuration Language

The Terraform configuration language is the main user interface for operating Terraform. Through configuration files, you can tell Terraform:
- Which plugins to install
- Which infrastructure to create
- Which data to retrieve

This configuration language is primarily based on HCL syntax, featuring simple configuration, strong readability, and JSON syntax compatibility.

## Language Overview

The main purpose of the Terraform language is to declare resources, which represent infrastructure objects. All other language features exist to make resource definitions more flexible and convenient.

### Basic Syntax Elements

The syntax of the Terraform language consists of the following basic elements:

```hcl
# Block configuration
resource "huaweicloud_vpc" "main" {
  # Parameter configuration
  cidr_block = var.base_cidr_block # Expression
}
```
1. **Blocks**: Serve as containers for other content, usually representing configuration for some kind of object.

2. **Arguments**: Assign values to names within blocks.

3. **Expressions**: Represent a value, which can be a literal value or reference other values, functions, combinations, etc.

### Declarative Features

The Terraform language is declarative, describing the desired target state rather than the steps to reach that target. Terraform will determine the order of operations based on implicit and explicit relationships between resources.

## Configuration Example

The following is a simple configuration example showing how to define a Huawei Cloud VPC network:

```hcl
# Configure Terraform
terraform {
  required_providers {
    huaweicloud = {
      source  = "huaweicloud/huaweicloud"
      version = "~> 1.47.1"
    }
  }
}

# Configure provider
provider "huaweicloud" {
  region     = "cn-north-4"
}

# Define variables
variable "vpc_cidr" {
  description = "VPC CIDR block"
  default     = "192.168.0.0/16"
}

# Create VPC
resource "huaweicloud_vpc" "example" {
  name = "terraform-vpc"
  cidr = var.vpc_cidr
}
```

## Deep Learning

To gain an in-depth understanding of all features of the Terraform configuration language, it is recommended to study the official documentation in the following order:

1. [Configuration Language Basics](https://developer.hashicorp.com/terraform/language)
   - File and directory structure
   - Syntax rules
   - Resources and data sources

2. [Resources and Data Sources](https://developer.hashicorp.com/terraform/language/resources)
   - Resource behavior
   - Dependencies
   - Meta-parameters

3. [Variables and Outputs](https://developer.hashicorp.com/terraform/language/values)
   - Variable definitions
   - Output values
   - Local values

4. [Expressions](https://developer.hashicorp.com/terraform/language/expressions)
   - Reference expressions
   - Operators
   - Conditional expressions

5. [Functions](https://developer.hashicorp.com/terraform/language/functions)
   - Built-in functions
   - Numeric functions
   - String functions
   - Collection functions

6. [Modules](https://developer.hashicorp.com/terraform/language/modules)
   - Module usage
   - Module creation
   - Module composition

Other content can be learned through the following links:
  - [Terraform Official Documentation](https://developer.hashicorp.com/terraform/language)
