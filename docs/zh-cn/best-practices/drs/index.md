# 简介

## 什么是数据复制服务（DRS）

数据复制服务（Data Replication Service，DRS）是华为云提供的一站式数据复制服务，致力于解决数据库上云、数据库迁移、数据库实时同步和数据库灾备等场景下的数据流转问题。DRS支持多种主流数据库引擎，包括MySQL、PostgreSQL、SQL Server、MongoDB、Oracle等，能够在保证业务连续性的前提下，实现数据在云上、云下以及跨云环境之间的高效复制。

DRS提供实时迁移、实时同步和实时灾备三大核心能力。实时迁移用于将本地或其他云上的数据库平滑迁移至华为云，支持在线迁移和断点续传；实时同步用于实现数据库之间的持续数据同步，适用于业务双活、读写分离、数据汇聚等场景；实时灾备用于构建跨可用区或跨地域的数据库容灾能力，保障业务的高可用性。

在使用DRS开展数据复制任务之前，需要先通过连接管理功能维护源数据库和目标数据库的接入信息，包括数据库类型、IP地址与端口、数据库用户名与密码、SSL配置以及驱动配置等。DRS连接是后续迁移、同步和灾备任务的基础，通过Terraform可以自动化地创建和管理这些连接，提升部署效率并降低人工配置出错的风险。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云数据复制服务（DRS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的DRS资源。

通过本章节的最佳实践，您可以学习到主要的DRS资源的部署流程，这些最佳实践将帮助您快速上手DRS的自动化部署，并为后续的数据复制任务管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署MongoDB分片连接](drs_connection_mongodb.md) - 介绍如何使用Terraform自动化部署MongoDB分片连接，包括DRS连接创建、主节点接入信息配置、分片节点接入信息配置、SSL配置和驱动配置。
* [部署RDS MySQL连接](drs_connection_rds_mysql.md) - 介绍如何使用Terraform自动化部署RDS MySQL连接，包括VPC创建、子网创建、安全组配置、RDS MySQL实例创建和DRS连接配置。

## 参考资料

- [华为云数据复制服务产品文档](https://support.huaweicloud.com/drs/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
