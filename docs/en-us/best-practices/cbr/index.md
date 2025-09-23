# Introduction

## What is Cloud Backup and Recovery (CBR)

Cloud Backup and Recovery (CBR) is a data protection service provided by Huawei Cloud, offering simple and easy-to-use backup services for both cloud and on-premises resources. When events such as virus intrusion, accidental deletion, or hardware/software failures occur, data can be restored to any backup point. CBR service supports multiple backup types, including cloud server backup, cloud disk backup, SFS Turbo backup, cloud desktop backup, etc., meeting data protection needs for different scenarios.

CBR service provides complete backup lifecycle management functionality, supporting automatic backup policies, incremental backup, cross-region replication, and other enterprise-grade features, with high reliability and security. Through CBR service, enterprises can easily achieve data protection, ensure business continuity and data security, and reduce data loss risks.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Cloud Backup and Recovery (CBR), helping you understand how to efficiently manage cloud backup resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for CBR resources. These best practices will help you quickly get started with automated CBR deployment and lay a solid foundation for subsequent backup policy management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Disk Type Vault](disk_vault.md) - Introduces how to use Terraform to automatically deploy a CBR disk type vault, including the creation of cloud disk volumes and vaults.
* [Deploy Server Type Vault](server_vault.md) - Introduces how to use Terraform to automatically deploy a CBR server type vault, including the creation of ECS instances, backup policies, and vaults.
* [Deploy SFS Turbo Type Vault](sfs_turbo_vault.md) - Introduces how to use Terraform to automatically deploy a CBR SFS Turbo type vault, including the creation of SFS Turbo file systems, backup policies, and vaults.

## Reference Materials

- [Huawei Cloud CBR Product Documentation](https://support.huaweicloud.com/cbr/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
