# 简介

## 什么是数据仓库服务（DWS）

数据仓库服务（Data Warehouse Service，DWS）是华为云提供的在线联机分析处理（OLAP）企业级数据仓库服务，基于华为云基础设施构建，面向海量数据分析场景，提供高性能、高可靠、易扩展的数据仓库能力。DWS 采用分布式大规模并行处理（MPP）架构，支持列式存储与向量化执行引擎，能够对万亿级数据实现快速查询与分析，帮助企业在云上快速构建数据仓库与商业智能（BI）应用。

DWS 提供标准数仓、实时数仓和云数仓等多种形态，兼容标准 SQL 与 PostgreSQL 生态，支持数据导入导出、数据共享、冷热数据分层存储等能力，并可与数据治理中心（DataArts Studio）、对象存储服务（OBS）、数据湖探索（DLI）等云服务协同，构建端到端的数据分析链路。

在运维层面，DWS 提供集群监控、事件订阅、告警通知、快照备份、容灾等企业级能力，帮助运维人员及时感知集群运行状态与异常事件，保障数据仓库服务的稳定性与连续性。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云数据仓库服务（DWS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的DWS资源。

通过本章节的最佳实践，您可以学习到主要的DWS资源的部署流程，这些最佳实践将帮助您快速上手DWS的自动化部署，并为后续的DWS集群管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署事件订阅](event_subscription.md) - 介绍如何使用Terraform自动化部署DWS事件订阅，包括VPC创建、子网与安全组配置、DWS集群创建、SMN主题与订阅创建以及事件订阅配置。
* [部署MySQL到DWS的实时数据同步](real_time_sync_mysql_to_dws.md) - 介绍如何使用Terraform自动化部署MySQL到DWS的实时数据同步基础设施，包括VPC、子网与安全组创建、RDS MySQL实例创建、DWS集群创建、DLI弹性资源池与队列创建以及增强型数据源连接关联。

## 参考资料

- [华为云数据仓库服务产品文档](https://support.huaweicloud.com/dws/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
