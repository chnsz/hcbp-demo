# 简介

## 什么是数据治理中心（DataArts Studio）

数据治理中心（DataArts Studio）是华为云提供的一站式数据运营治理平台，具有数据全生命周期管理和智能数据管理能力，支持行业知识库智能化建设。DataArts Studio支持对接华为云数据湖与数据库云服务，帮助企业快速构建从数据接入到数据分析的端到端智能数据系统，消除数据孤岛，统一数据标准，加速数据变现。

DataArts Studio提供数据集成、数据开发、数据架构、数据质量、数据目录、数据服务和数据安全等功能模块。其中DataArts Factory（数据开发）提供全托管的大数据调度与开发环境，支持DLI SQL等多种脚本类型，便于用户进行数据管理、作业开发与运维监控。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云数据治理中心（DataArts Studio）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的DataArts资源。

通过本章节的最佳实践，您可以学习到主要的DataArts Studio资源与关联DLI资源的部署流程，这些最佳实践将帮助您快速上手DataArts的自动化部署，并为后续的数据开发脚本、作业执行和运维管理工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署DataArts Factory脚本执行](factory_script_execute.md) - 介绍如何使用Terraform自动化部署DataArts Factory脚本及脚本执行，包括工作空间查询、DLI数据连接查询、DLI数据库与表、Factory脚本及脚本执行的创建。

## 参考资料

- [华为云DataArts Studio产品文档](https://support.huaweicloud.com/dataartsstudio/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
