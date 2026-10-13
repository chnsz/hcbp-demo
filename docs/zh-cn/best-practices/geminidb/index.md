# 简介

## 什么是云原生多模NoSQL数据库（GeminiDB）

GeminiDB是华为云提供的云原生多模NoSQL数据库服务，基于计算存储分离架构构建，兼容Cassandra、MongoDB、InfluxDB、Redis等多种主流NoSQL接口，能够满足不同业务场景下的数据存储与访问需求。GeminiDB将计算节点与存储节点解耦，支持存储按需扩展与计算节点秒级弹性伸缩，帮助企业在业务负载波动时保持稳定的性能表现。

GeminiDB具备高可靠、高可用、高性能等特性，通过多副本冗余、自动故障切换和备份恢复等机制保障数据安全与业务连续性。同时，GeminiDB提供丰富的运维能力，包括监控告警、参数模板、备份管理等，帮助用户降低数据库运维成本，专注于业务开发与创新。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云云原生多模NoSQL数据库（GeminiDB）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的GeminiDB资源。

通过本章节的最佳实践，您可以学习到主要的GeminiDB资源的部署流程，这些最佳实践将帮助您快速上手GeminiDB的自动化部署，并为后续的GeminiDB管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署GeminiDB Cassandra实例](geminidb_cassandra_instance.md) - 介绍如何使用Terraform自动化部署GeminiDB Cassandra实例，包括VPC创建、子网创建、安全组配置、规格查询、随机密码生成和实例备份管理。

## 参考资料

- [华为云GeminiDB产品文档](https://support.huaweicloud.com/geminidb/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
