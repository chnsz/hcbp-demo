# Introduction

## What is Relational Database Service (RDS)

Relational Database Service (RDS) is a highly available, high-performance, and easily scalable relational database cloud service provided by Huawei Cloud, supporting multiple database engines such as MySQL, PostgreSQL, SQL Server, MariaDB, etc. RDS provides automatic backup, monitoring alerts, elastic scaling, read-write separation, and other functions, meeting the database requirements of enterprise applications. RDS features high availability, data security, and ease of use, supporting seamless integration with ECS, CCE, FunctionGraph, and other services.

As one of Huawei Cloud's core database services, RDS supports multiple database engines and deployment modes, meeting complex database management scenario requirements. RDS provides flexible instance configuration and database management capabilities, supporting advanced functions such as automatic backup, monitoring alerts, and permission management.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Relational Database Service (RDS), helping you understand how to efficiently manage cloud database resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for database resources. These best practices will help you quickly get started with automated RDS deployment and lay a solid foundation for subsequent RDS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy MySQL Single Instance](mysql_single_instance.md) - Introduces how to use Terraform to automatically deploy RDS MySQL single instances, including VPC network, security group, RDS instance, database account, and database creation, supporting complete MySQL database management functions.
* [Deploy MySQL Single Instance with EIP](mysql_single_instance_with_eip.md) - Introduces how to use Terraform to automatically deploy RDS MySQL single instances with EIP binding, including VPC network, security group, RDS instance, EIP, and EIP binding creation, supporting MySQL database functions with public network access.
* [Deploy MySQL Read Replica Instance](mysql_read_replica_instance.md) - Introduces how to use Terraform to automatically deploy RDS MySQL primary-standby instances and read replica instances, including VPC network, security group, RDS instance, and read replica instance creation, supporting high availability and read-write separation functions.
* [Deploy PostgreSQL HA Instance](postgresql_ha_instance.md) - Introduces how to use Terraform to automatically deploy RDS PostgreSQL primary-standby instances, including VPC network, security group, RDS instance, PostgreSQL account, database, schema, and backup creation, supporting high-availability PostgreSQL database functions.
* [Deploy SQL Server Single Instance](sqlserver_single_instance.md) - Introduces how to use Terraform to automatically deploy RDS SQL Server single instances, including VPC network, security group, and RDS instance creation, supporting complete SQL Server database management functions.

## Reference Materials

- [Huawei Cloud Relational Database Service Product Documentation](https://support.huaweicloud.com/rds/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
