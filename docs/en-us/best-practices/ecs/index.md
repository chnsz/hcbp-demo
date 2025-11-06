# Introduction

## What is Elastic Cloud Server (ECS)

Elastic Cloud Server (ECS) is a fundamental computing component composed of CPU, memory, operating system, and cloud disks. As one of Huawei Cloud's core services, ECS provides scalable, high-performance computing resources, supporting multiple specification families and operating systems to meet the needs of different application scenarios.

Through ECS, you can quickly deploy applications and services in the cloud just like using your own local servers, enjoying the advantages of cloud computing such as elastic scaling, pay-as-you-go, high reliability, and security. Huawei Cloud ECS supports seamless integration with VPC, security groups, cloud disks, and other services, providing you with complete cloud solutions.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Elastic Cloud Server (ECS), helping you understand how to efficiently manage cloud computing resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for Elastic Cloud Server resources. These best practices will help you quickly get started with automated ECS deployment and lay a solid foundation for subsequent ECS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Basic Instance](simple_instance.md) - Introduces how to use Terraform to automatically deploy a basic ECS instance, including VPC, subnet, and security group creation.
* [Deploy Instance with EIP](instance_with_eip.md) - Introduces how to use Terraform to automatically deploy ECS instances with EIP binding, including network environment creation, security group configuration, instance creation, and EIP binding.
* [Deploy Instance with Provisioner Remote Login](instance_with_provisioner.md) - Introduces how to use Terraform to automatically deploy ECS instances with provisioner remote login, including key pair creation, network environment configuration, EIP binding, and remote command execution.
* [Deploy Instance with UserData Script Execution](instance_with_userdata.md) - Introduces how to use Terraform to automatically deploy ECS instances with UserData script execution, including network environment creation, security group configuration, key pair creation, instance creation, and script execution.

## Reference Materials

- [Huawei Cloud ECS Product Documentation](https://support.huaweicloud.com/ecs/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
