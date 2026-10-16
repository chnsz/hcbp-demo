# 简介

## 什么是GeminiDB

GeminiDB是华为云提供的云原生多模NoSQL数据库服务，基于计算与存储分离的架构设计，兼容Cassandra、MongoDB、Redis、InfluxDB等多种主流NoSQL接口。GeminiDB面向海量数据的高并发读写场景，提供高可用、高可靠、弹性扩展的数据库能力，帮助企业以更低的运维成本构建现代化的数据存储底座。

GeminiDB采用存算分离与分布式共享存储架构，支持秒级弹性扩容和按需扩缩容，能够有效应对业务负载的波动。同时，GeminiDB提供自动备份、故障自动切换、监控告警等企业级能力，保障数据的安全性与业务的连续性。

通过GeminiDB，用户无需自行搭建和维护NoSQL数据库集群，即可获得稳定、高性能的多模数据库服务，将更多精力投入到业务开发与创新之中。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云GeminiDB的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的GeminiDB资源。

通过本章节的最佳实践，您可以学习到主要的GeminiDB资源的部署流程，这些最佳实践将帮助您快速上手GeminiDB的自动化部署，并为后续的GeminiDB管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署GeminiDB Cassandra实例](geminidb_cassandra_instance.md) - 介绍如何使用Terraform自动化部署GeminiDB Cassandra实例，包括VPC与子网创建、可用区与规格查询、安全组配置、实例创建和实例备份。

## 参考资料

- [华为云GeminiDB产品文档](https://support.huaweicloud.com/geminidb/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
