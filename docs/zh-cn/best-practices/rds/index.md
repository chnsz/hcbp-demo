# 关系型数据库服务（RDS）最佳实践

## 概述

关系型数据库服务（Relational Database Service，RDS）是华为云提供的高可用、高性能、易扩展的关系型数据库云服务，支持MySQL、PostgreSQL、SQL Server、MariaDB等多种数据库引擎。RDS提供自动备份、监控告警、弹性扩容、读写分离等功能，能够满足企业级应用的数据库需求。RDS具备高可用性、数据安全性和易用性等特点，支持与ECS、CCE、FunctionGraph等服务的无缝集成。

作为华为云的核心数据库服务之一，RDS支持多种数据库引擎和部署模式，能够满足复杂的数据库管理场景需求。RDS提供灵活的实例配置和数据库管理能力，支持自动备份、监控告警、权限管理等高级功能。

本章节提供了使用Terraform自动化部署和管理华为云关系型数据库服务（RDS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的数据库资源。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署MySQL单机实例](mysql_single_instance.md) - 介绍如何使用Terraform自动化部署RDS MySQL单机实例，包括VPC网络、安全组、RDS实例、数据库账户和数据库的创建，支持完整的MySQL数据库管理功能。
* [部署绑定EIP的MySQL单机实例](mysql_single_instance_with_eip.md) - 介绍如何使用Terraform自动化部署绑定EIP的RDS MySQL单机实例，包括VPC网络、安全组、RDS实例、EIP和EIP绑定的创建，支持公网访问的MySQL数据库功能。
* [部署PostgreSQL主备实例](postgresql_ha_instance.md) - 介绍如何使用Terraform自动化部署RDS PostgreSQL主备实例，包括VPC网络、安全组、RDS实例、PostgreSQL账户、数据库、Schema和备份的创建，支持高可用的PostgreSQL数据库功能。

## 参考资料

- [华为云关系型数据库服务产品文档](https://support.huaweicloud.com/rds/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
