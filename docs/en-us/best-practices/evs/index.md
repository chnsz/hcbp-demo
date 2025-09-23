# Introduction

## What is Elastic Volume Service (EVS)

Elastic Volume Service (EVS) is a high-performance, highly reliable, and scalable block storage service provided by Huawei Cloud, providing persistent storage for ECS instances. EVS supports multiple storage types, including SSD, SAS, SATA, etc., meeting storage requirements for different business scenarios.

Through EVS, you can create, mount, unmount, and manage cloud volumes, providing persistent storage capabilities for ECS instances. Huawei Cloud EVS supports snapshot functionality, enabling data backup creation, and supports snapshot group functionality for consistent backup of multiple cloud volumes. Huawei Cloud EVS supports seamless integration with ECS, VPC, and other services, providing you with complete storage solutions.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Elastic Volume Service (EVS), helping you understand how to efficiently manage cloud storage resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for cloud volume resources. These best practices will help you quickly get started with automated EVS deployment and lay a solid foundation for subsequent EVS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Cloud Volume](volume.md) - Introduces how to use Terraform to automatically deploy EVS cloud volumes, including availability zone selection, storage type configuration, and performance parameter settings.
* [Deploy Disk Snapshot](snapshot.md) - Introduces how to use Terraform to automatically deploy EVS disk snapshots, including cloud volume creation and snapshot configuration.
* [Deploy Disk Snapshot Group](snapshot_group.md) - Introduces how to use Terraform to automatically deploy EVS disk snapshot groups, including ECS instance creation, cloud volume creation, mounting, and disk snapshot group configuration.

## Reference Materials

- [Huawei Cloud EVS Product Documentation](https://support.huaweicloud.com/evs/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
