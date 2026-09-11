# 简介

## 什么是数据湖探索（DLI）

数据湖探索（Data Lake Insight，DLI）是华为云提供的大数据计算与分析服务，支持SQL、Flink和Spark等多种计算引擎，帮助用户快速构建数据湖分析业务。DLI提供完全托管的计算资源，用户无需关心底层基础设施的运维，即可轻松处理海量数据，实现数据洞察与业务创新。

DLI支持多种数据源接入，包括对象存储服务（OBS）、数据接入服务（DIS）和关系型数据库等，并提供丰富的作业类型，如SQL作业、Flink作业和Spark作业，满足批处理、流处理和交互式查询等多种场景需求。同时，DLI提供弹性资源池和队列管理能力，用户可以根据业务负载灵活调整计算资源，实现成本与性能的平衡。

DLI还提供完善的数据安全与权限管理功能，支持细粒度的访问控制和数据加密，保障企业数据的安全合规。通过DLI，企业可以快速构建数据湖分析平台，挖掘数据价值，驱动业务决策。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云数据湖探索（DLI）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的DLI资源。

通过本章节的最佳实践，您可以学习到主要的DLI资源的部署流程，这些最佳实践将帮助您快速上手DLI的自动化部署，并为后续的弹性资源池、队列和Flink作业管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署Flink Jar作业](flink_jar_job.md) - 介绍如何使用Terraform自动化部署Flink Jar作业，包括弹性资源池创建、通用队列配置和作业参数设置。
* [部署Flink OpenSource SQL作业](flink_opensource_sql_job.md) - 介绍如何使用Terraform自动化部署Flink OpenSource SQL作业，包括弹性资源池创建、通用队列配置和作业参数设置。
* [部署队列公网连通](queue_public_network_connectivity.md) - 介绍如何使用Terraform自动化部署DLI队列公网连通，包括弹性资源池与队列创建、VPC与子网配置、增强型跨源连接关联、弹性公网IP与NAT网关创建和SNAT规则配置。

## 参考资料

- [华为云数据湖探索产品文档](https://support.huaweicloud.com/dli/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
