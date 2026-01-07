# 简介

## 什么是API网关（APIG）

API网关（API Gateway）是为企业和开发者提供的高性能、高可用、高安全的云原生网关服务。它能快速将企业服务能力包装成标准API接口，帮助您轻松构建、管理和部署任意规模的API，并支持API上架云商店进行售卖。借助API网关，您可以简单、快速、低成本、低风险地实现内部系统集成和业务能力开放。

API网关提供完整的API生命周期管理功能，支持多种安全认证和防护机制，具备云原生网关能力，可简化架构并降低部署和运维成本。通过API网关，企业可以快速实现业务能力开放，构建API生态，实现业务价值最大化。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云API网关（APIG）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的API网关资源。

通过本章节的最佳实践，您可以学习到主要的API网关资源的部署流程，这些最佳实践将帮助您快速上手API网关的自动化部署，并为后续的API管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [创建Kafka转发插件](kafka_forward_plugin.md) - 介绍如何使用Terraform自动化创建API网关的Kafka转发插件，实现API请求的异步转发和消息队列集成。
* [创建代理缓存插件](proxy_cache_plugin.md) - 介绍如何使用Terraform自动化创建API网关的代理缓存插件，实现API响应数据的缓存管理。
* [部署带有自定义认证的API](function_authorizer.md) - 介绍如何使用Terraform自动化部署带有自定义认证的API以及如何使用FunctionGraph函数实现API的前端认证。

## 参考资料

- [华为云API网关产品文档](https://support.huaweicloud.com/apig/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
