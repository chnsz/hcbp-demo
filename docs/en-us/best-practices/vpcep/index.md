# Introduction

## What is VPC Endpoint (VPCEP)

VPC Endpoint (VPCEP) is a VPC internal resource mutual access service provided by Huawei Cloud, supporting the creation of endpoints and endpoint services within VPCs to achieve private network access to VPC resources. VPCEP service supports cross-VPC private network access, avoiding public network access, improving access security and stability, reducing network latency and costs.

VPCEP service provides complete endpoint and endpoint service lifecycle management functions, supporting the creation and configuration of resources such as endpoint services and endpoints, with high reliability and security. Through VPCEP service, enterprises can easily achieve cross-VPC private network access, building secure and efficient hybrid cloud architectures.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud VPC Endpoint (VPCEP), helping you understand how to efficiently manage cloud VPCEP resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for VPCEP resources. These best practices will help you quickly get started with automated VPCEP deployment and lay a solid foundation for subsequent endpoint service, endpoint management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Service](service.md) - Introduces how to use Terraform to automatically deploy endpoint services, including availability zone query, ECS flavor query, image query, VPC, subnet, security group, ECS instance, and endpoint service creation.
* [Deploy Endpoint](endpoint.md) - Introduces how to use Terraform to automatically deploy endpoints, including availability zone query, ECS flavor query, image query, VPC, subnet, security group, ECS instance, endpoint service, and endpoint creation.

## Reference Materials

- [Huawei Cloud VPCEP Product Documentation](https://support.huaweicloud.com/intl/en-us/vpcep/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
