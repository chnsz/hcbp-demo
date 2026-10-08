# Introduction

## What is GaussDB

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud. Built on Huawei's years of database technology accumulation and software-hardware collaborative optimization, it supports both centralized and distributed deployment modes, meeting diverse requirements ranging from small and medium-sized businesses to massive-data core systems.

GaussDB is fully compatible with mainstream database ecosystems, supporting standard SQL as well as advanced features such as stored procedures, triggers, and user-defined functions. It also provides enterprise-grade capabilities including high availability architecture, automatic backup and recovery, disaster recovery switchover, read/write splitting, and elastic scaling, helping users quickly build a stable and reliable data foundation on the cloud.

In terms of security and operations, GaussDB provides client access authentication (HBA) configuration, fine-grained permission management, transparent encryption, and audit logs, and supports comprehensive observability through services such as Cloud Eye and Log Tank Service, helping enterprises reduce database O&M costs while ensuring data security and compliance.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud GaussDB, helping you understand how to efficiently manage cloud GaussDB resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for GaussDB resources. These best practices will help you quickly get started with automated GaussDB deployment and lay a solid foundation for subsequent GaussDB management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Client Access Authentication Configuration Restore](client_auth_config_restore.md) - Introduces how to use Terraform to automatically deploy client access authentication configuration restore, including GaussDB instance specification, history record version selection, and default configuration restoration.

## Reference Materials

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
