# Introduction

## What is Object Storage Service (OBS)

Object Storage Service (OBS) is a highly available, highly reliable, high-performance, secure, and low-cost object storage service provided by Huawei Cloud. OBS provides massive, secure, highly reliable, and low-cost data storage capabilities, supporting multiple storage types, including standard storage, infrequent access storage, archive storage, etc., meeting storage requirements for different business scenarios.

Through OBS, you can store and manage any type of data, including documents, images, videos, audio, etc. Huawei Cloud OBS supports multiple security features, including server-side encryption, access control, audit logs, etc., providing enterprises with secure and reliable data storage solutions. Huawei Cloud OBS supports seamless integration with ECS, FunctionGraph, CDN, and other services, providing you with complete cloud storage solutions.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Object Storage Service (OBS), helping you understand how to efficiently manage cloud storage resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for object storage resources. These best practices will help you quickly get started with automated OBS deployment and lay a solid foundation for subsequent OBS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy KMS Encrypted Bucket](kms_encrypted_bucket.md) - Introduces how to use Terraform to automatically deploy OBS KMS encrypted buckets, including KMS key creation and OBS bucket configuration.
* [Deploy Bucket Static Website Hosting Configuration](static_website_hosting.md) - Introduces how to use Terraform to automatically deploy OBS bucket static website hosting configuration, including KMS key creation, OBS bucket configuration, website configuration, and access policy settings.
* [Deploy Bucket Object Uploaded with Static Text](object_upload_with_content.md) - Introduces how to use Terraform to automatically deploy bucket objects uploaded with static text, including KMS key creation, OBS bucket configuration, and object upload.
* [Deploy Bucket Object Uploaded with Encrypted Text](object_upload_with_encryption.md) - Introduces how to use Terraform to automatically deploy bucket objects uploaded with encrypted text, including KMS key creation, OBS bucket configuration, file compression, and encrypted upload.
* [Deploy Bucket Object Uploaded with Local File](object_upload_with_source.md) - Introduces how to use Terraform to automatically deploy bucket objects uploaded with local files, including KMS key creation, OBS bucket configuration, file compression, and upload.

## Reference Materials

- [Huawei Cloud OBS Product Documentation](https://support.huaweicloud.com/obs/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
