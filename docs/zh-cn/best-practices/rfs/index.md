# 简介

## 什么是资源编排服务（RFS）

资源编排服务（Resource Formation Service，RFS）是华为云提供的基础设施即代码（Infrastructure as Code，IaC）服务，帮助用户以代码的方式定义、编排和自动化部署云上资源。通过模板化的资源描述，RFS 能够将复杂的云资源组合封装为可复用的模板，实现基础设施的标准化交付与一致性管理，降低手工配置带来的错误风险。

RFS 支持模板的创建、更新、删除以及执行计划的预览，用户可以在执行前清晰地了解资源变更内容，从而安全、可控地完成资源编排。同时，RFS 提供私有模块能力，允许用户将一组模板封装为标准化模块，并通过模块版本实现模板的版本化管理，便于在团队与项目之间共享和复用基础设施代码。

借助 RFS，企业可以将基础设施的部署流程沉淀为可维护、可追溯的代码资产，配合版本管理与自动化执行，为后续的资源治理、合规审计和持续交付奠定坚实基础。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云资源编排服务（RFS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的RFS资源。

通过本章节的最佳实践，您可以学习到主要的RFS资源的部署流程，这些最佳实践将帮助您快速上手RFS的自动化部署，并为后续的模板、模块管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署私有模块及模块版本](private_module_with_version.md) - 介绍如何使用Terraform自动化部署私有模块及模块版本，包括私有模块创建、模块版本配置和OBS模块包引用。
* [部署基于FunctionGraph后端的私有Provider](private_provider_with_functiongraph.md) - 介绍如何使用Terraform自动化部署基于FunctionGraph后端的私有Provider，包括FunctionGraph函数创建、私有Provider创建和私有Provider版本发布。

## 参考资料

- [华为云资源编排服务（RFS）产品文档](https://support.huaweicloud.com/rfs/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
