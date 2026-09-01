# Introduction

## What is Database Security Service (DBSS)

Database Security Service (DBSS) provides database security audit, database security encryption, and database security O&M capabilities. Database audit records user access behaviors to databases in real time, generates fine-grained audit reports, and sends real-time alarms for risky and attack behaviors.

Database security audit uses bypass deployment and supports auditing Huawei Cloud RDS databases and self-built databases on ECS/BMS. It enables flexible auditing without affecting user services, helping enterprises locate and hold accountable internal violations and improper operations to protect data assets.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Database Security Service (DBSS), helping you understand how to efficiently manage DBSS resources on the cloud using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DBSS resources. These best practices will help you quickly get started with automated DBSS deployment and lay a solid foundation for subsequent management and operations of database security audit instances, ECS self-built database auditing, and RDS database auditing.

## Best Practices List

This section contains the following best practices:

* [Deploy Audit ECS Database](audit_ecs_database.md) - Introduces how to use Terraform to automatically deploy a DBSS audit instance and add an ECS self-built database, including creating VPC, subnet, security group, and DBSS instance, as well as configuring self-built database audit information.
* [Deploy Audit RDS Database](audit_rds_database.md) - Introduces how to use Terraform to automatically deploy a DBSS audit instance and add an RDS database, including creating VPC, subnet, security group, RDS instance, and DBSS instance, as well as configuring RDS database audit information.

## Reference Materials

- [Huawei Cloud DBSS Product Documentation](https://support.huaweicloud.com/dbss/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
