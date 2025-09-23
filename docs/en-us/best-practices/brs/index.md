# Introduction

## What is Business Recovery Service (BRS)

Business Recovery Service (BRS, named as SDRS before) is a disaster recovery service for Elastic Cloud Server (ECS) and Elastic Volume Service (EVS). Through host-level replication, data redundancy, and cache acceleration technologies, it provides users with high-level data reliability and business continuity, known as Business Recovery Service.

Business Recovery Service helps protect business applications by replicating Elastic Cloud Server data and configuration information to disaster recovery sites, allowing business applications to start and run normally on disaster recovery site cloud servers during production site cloud server downtime, thereby improving business continuity. BRS supports multiple disaster recovery modes, including same-city disaster recovery, cross-region disaster recovery, etc., and provides disaster recovery drill functionality, allowing you to regularly verify the feasibility of disaster recovery solutions.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Business Recovery Service (BRS), helping you understand how to efficiently manage cloud disaster recovery resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for disaster recovery resources. These best practices will help you quickly get started with automated Business Recovery Service deployment and lay a solid foundation for subsequent protection group, protected instance, and disaster recovery drill management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Disaster Recovery Drill](disaster_recovery_drill.md) - Introduces how to use Terraform to automatically deploy BRS disaster recovery drill environment, including the creation of protection groups, protected instances, and disaster recovery drills.
* [Deploy Protection Group](protection_group.md) - Introduces how to use Terraform to automatically deploy BRS protection group environment, including the creation of protection groups.
* [Deploy Protected Instance](protected_instance.md) - Introduces how to use Terraform to automatically deploy BRS protected instance environment, including the creation of protection groups and protected instances.

## Reference Materials

- [Huawei Cloud BRS Product Documentation](https://support.huaweicloud.com/sdrs/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
