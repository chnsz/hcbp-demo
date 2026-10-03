# 简介

## 什么是GeminiDB

GeminiDB是华为云提供的多模NoSQL数据库服务，兼容Cassandra、MongoDB、InfluxDB、Redis等多种主流NoSQL引擎协议，具备高可用、高可靠、弹性扩展、安全可信等特性，能够满足海量数据存储、高并发读写以及多样化数据模型等业务需求。

GeminiDB采用存算分离架构与分布式集群设计，支持计算节点与存储容量的独立弹性伸缩，并提供自动备份、故障切换、监控告警等企业级能力，帮助用户在云上快速构建稳定可靠的NoSQL数据库服务。

通过GeminiDB，用户可以按需选择不同的引擎与规格，灵活应对物联网、游戏、社交、金融风控等场景下的数据存储与访问挑战，降低数据库运维成本，提升业务连续性与数据安全性。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云GeminiDB的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的GeminiDB资源。

通过本章节的最佳实践，您可以学习到主要的GeminiDB资源的部署流程，这些最佳实践将帮助您快速上手GeminiDB的自动化部署，并为后续的GeminiDB管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署GeminiDB Cassandra实例](geminidb_cassandra_instance.md) - 介绍如何使用Terraform自动化部署GeminiDB Cassandra实例，包括VPC创建、子网创建、安全组配置、实例规格查询、密码生成、备份策略和实例备份管理。

## 参考资料

- [华为云GeminiDB产品文档](https://support.huaweicloud.com/geminidb/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
