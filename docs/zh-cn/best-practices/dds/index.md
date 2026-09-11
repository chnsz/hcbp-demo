# 简介

## 什么是文档数据库服务（DDS）

文档数据库服务（Document Database Service，简称DDS）是华为云提供的高性能、高可靠、高安全性的分布式文档数据库服务，完全兼容MongoDB协议，支持副本集、集群和单节点等多种部署架构，适用于多种业务场景。DDS提供自动备份、监控告警、弹性伸缩等企业级功能，帮助用户轻松构建和管理文档数据库，降低运维成本，提升业务效率。

DDS支持多种数据库版本和存储引擎，提供丰富的实例规格和节点类型，满足不同规模和性能要求的业务需求。同时，DDS提供数据备份与恢复、容灾切换、安全审计等能力，保障数据的安全性和业务的连续性。

通过DDS，用户可以快速部署高可用的文档数据库，实现数据的可靠存储和高效访问，支撑互联网、物联网、游戏、金融等行业的核心业务。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云文档数据库服务（DDS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的DDS资源。

通过本章节的最佳实践，您可以学习到主要的DDS资源的部署流程，这些最佳实践将帮助您快速上手DDS的自动化部署，并为后续的DDS实例、网络、安全组等管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署DDS实例绑定弹性公网IP](dds_associate_eip.md) - 介绍如何使用Terraform创建DDS实例并绑定弹性公网IP，实现公网访问。
* [部署DDS实例关联LTS](dds_associate_lts.md) - 介绍如何使用Terraform自动化部署DDS实例关联LTS，包括可用分区（data.）、DDS规格（data.）、VPC创建、子网配置和安全组配置。
* [部署DDS实例绑定NAT网关](dds_associate_nat.md) - 介绍如何使用Terraform自动化部署DDS实例绑定NAT网关，包括查询可用分区（data.）、查询DDS实例信息（data.）、VPC创建、子网配置和安全组配置。
* [部署DDS实例备份](dds_backup.md) - 介绍如何使用Terraform自动化部署DDS实例备份，包括可用分区列表（data.）、VPC创建、子网配置、安全组配置和实例配置。

## 参考资料

- [华为云文档数据库服务产品文档](https://support.huaweicloud.com/dds/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
