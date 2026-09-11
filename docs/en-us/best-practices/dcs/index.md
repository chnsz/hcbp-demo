# Introduction

## What is Distributed Cache Service (DCS)

Distributed Cache Service (DCS) is a high-performance, highly available in-memory database service provided by Huawei Cloud, supporting mainstream cache engines such as Redis and Memcached. DCS service provides multiple instance specifications and deployment modes, including single-node, master-standby, and cluster, meeting cache requirements for different scales and scenarios.

DCS service provides complete cache lifecycle management functionality, supporting enterprise-level features such as automatic backup, monitoring alerts, and parameter tuning, with high reliability and security. Through DCS service, enterprises can easily implement application scenarios such as data caching, session storage, and message queues, improving application performance and user experience.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Distributed Cache Service (DCS), helping you understand how to efficiently manage cloud cache resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DCS resources. These best practices will help you quickly get started with automated DCS deployment and lay a solid foundation for subsequent cache management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Master-Standby Redis Instance](redis_ha_instance.md) - Introduces how to use Terraform to automatically deploy DCS master-standby Redis instances, including VPC creation, instance configuration, backup policy, and whitelist management.
* [Deploy Redis Account Management](redis_account.md) - Introduces how to use Terraform to automatically deploy DCS Redis instances and accounts, including VPC creation, subnet, instance configuration, and account permission management.
* [Deploy Redis Background Task Deletion](redis_background_task_delete.md) - Introduces how to use Terraform to delete specified background tasks of a DCS Redis instance, suitable for cleaning completed or abnormal background tasks.
* [Deploy Redis Big Key Analysis](redis_bigkey_analysis.md) - Introduces how to use Terraform to automatically deploy a DCS Redis instance and create a big key analysis task, including VPC creation, subnet, instance configuration, and big key analysis.
* [Deploy Redis Center Task Deletion](redis_center_task_delete.md) - Introduces how to use Terraform to automatically deploy Redis Center Task Deletion, including DCS Center Task Deletion.
* [Deploy Redis Custom Template](redis_custom_template.md) - Introduces how to use Terraform to automatically deploy Redis Custom Template, including DCS Custom Template.
* [Deploy Redis Data Synchronization](redis_data_sync.md) - Introduces how to use Terraform to automatically deploy Redis Data Synchronization, including Availability Zones (data.), DCS Flavors (data.), VPC creation, subnet configuration, and security group configuration.
* [Deploy Redis Diagnosis Task](redis_diagnosis_task.md) - Introduces how to use Terraform to automatically deploy Redis Diagnosis Task, including DCS Redis Diagnosis Task.
* [Deploy Redis Expired Key Scan](redis_expired_key_scan.md) - Introduces how to use Terraform to automatically deploy Redis Expired Key Scan, including Availability Zones (data.), DCS Flavors (data.), VPC creation, subnet configuration, and Random Password (random_password).
* [Deploy Redis Instance All Sessions Kill](redis_all_sessions_kill.md) - Introduces how to use Terraform to automatically deploy a DCS Redis instance and kill all sessions, including VPC creation, instance configuration, and session cleanup.
* [Deploy Redis Instance Backup](redis_backup.md) - Introduces how to use Terraform to automatically deploy a DCS Redis instance and create a manual backup, including VPC creation, instance configuration, and backup resource management.
* [Deploy Single-Node Redis Instance](redis_single_instance.md) - Introduces how to use Terraform to automatically deploy DCS single-node Redis instances, including VPC creation, instance configuration, and basic network setup.

## Reference Materials

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
