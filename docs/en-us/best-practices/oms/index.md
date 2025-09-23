# Introduction

## What is Object Storage Migration Service (OMS)

Object Storage Migration Service (OMS) is a one-stop data migration service provided by Huawei Cloud, helping users quickly, securely, and efficiently migrate data from other cloud service providers or local environments to Huawei Cloud Object Storage Service (OBS). OMS service supports multiple data sources, including mainstream cloud storage services such as AWS S3, Alibaba Cloud OSS, Tencent Cloud COS, as well as local file systems.

Through OMS, you can achieve seamless data migration, supporting incremental synchronization, resumable transfer, data verification, and other functions, ensuring the integrity and consistency of data migration. OMS service provides graphical interfaces and API interfaces, supporting management and monitoring of large-scale data migration tasks, providing enterprises with reliable data migration solutions. Huawei Cloud OMS supports seamless integration with OBS, KMS, and other services, providing you with a complete storage migration ecosystem.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Object Storage Migration Service (OMS), helping you understand how to efficiently manage cloud migration resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for object storage migration resources. These best practices will help you quickly get started with automated OMS deployment and lay a solid foundation for subsequent OMS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Object Migration Through Task Groups](group_migration.md) - Introduces how to use Terraform to automatically deploy OMS task group migration tasks, including KMS key creation, OBS bucket configuration, object upload, bucket policy settings, and migration task group creation.
* [Deploy Object Migration Through Tasks](task_migration.md) - Introduces how to use Terraform to automatically deploy OMS task migration, including KMS key creation, OBS bucket configuration, object upload, bucket policy settings, and migration task creation.

## Reference Materials

- [Huawei Cloud OMS Product Documentation](https://support.huaweicloud.com/oms/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
