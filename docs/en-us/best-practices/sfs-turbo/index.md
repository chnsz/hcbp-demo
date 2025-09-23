# Introduction

## What is Scalable File Service Turbo (SFS Turbo)

Scalable File Service Turbo (SFS Turbo) is a high-performance file storage service provided by Huawei Cloud, specifically designed for high-performance computing (HPC) and AI/ML workloads. SFS Turbo provides high IOPS, low-latency file storage capabilities, supporting NFS and CIFS protocols, meeting the requirements of large-scale data processing, machine learning training, scientific computing, and other scenarios. SFS Turbo features high availability, elastic scaling, and ease of use, supporting seamless integration with OBS object storage.

As one of Huawei Cloud's core storage services, SFS Turbo supports seamless integration with ECS, CCE, FunctionGraph, and other services, meeting complex file storage scenario requirements. SFS Turbo provides flexible storage configuration and OBS target configuration capabilities, supporting advanced functions such as automatic data export and permission management.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Scalable File Service (SFS Turbo), helping you understand how to efficiently manage cloud file storage resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for file storage resources. These best practices will help you quickly get started with automated SFS Turbo deployment and lay a solid foundation for subsequent SFS Turbo management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy File System](file_system.md) - Introduces how to use Terraform to automatically deploy SFS Turbo file systems, including VPC network, security group, and SFS Turbo file system creation, supporting high-performance file storage and multiple protocol access.
* [Deploy OBS Target Configuration](obs_target_configuration.md) - Introduces how to use Terraform to automatically deploy SFS Turbo OBS target configuration, including VPC network, SFS Turbo file system, and OBS target configuration creation, supporting automatic data export and permission management functions.
* [Deploy Permission Rules](permission_rule.md) - Introduces how to use Terraform to automatically deploy SFS Turbo permission rules, including VPC network, security group, SFS Turbo file system, and permission rule creation, supporting access control and security isolation functions.

## Reference Materials

- [Huawei Cloud Scalable File Service Product Documentation](https://support.huaweicloud.com/sfsturbo/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
