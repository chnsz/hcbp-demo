# 简介

## 什么是函数工作流（FunctionGraph）

函数工作流（FunctionGraph）是一项基于事件驱动的无服务器计算服务。它提供了零代码、低代码、自定义代码三种开发方式，支持多种编程语言，让您可以无需管理服务器等基础设施，只需编写业务函数代码并设置运行条件，即可弹性、可靠地运行任务。通过函数工作流，您可以快速构建灵活的数据处理流程和应用集成方案。

作为华为云的核心计算服务之一，FunctionGraph支持多种触发器类型，包括定时触发器、OBS触发器、SMN触发器、APIG触发器、DMS触发器等，能够满足不同业务场景的需求。FunctionGraph提供弹性伸缩、按需付费、高可靠性等云计算优势，让您专注于业务逻辑开发，无需关心底层基础设施管理。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云函数工作流（FunctionGraph）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的无服务器计算资源。

通过本章节的最佳实践，您可以学习到主要的无服务器计算资源的部署流程，这些最佳实践将帮助您快速上手函数工作流的自动化部署，并为后续的FunctionGraph管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署CTS触发器](cts_trigger.md) - 通过CTS触发器实现云资源操作监控和响应，支持安全审计、合规监控、自动化运维等功能，提供完整的Terraform配置和参数说明。
* [部署EG触发器](eg_trigger.md) - 通过EG触发器实现事件驱动的图像处理，支持OBS文件上传时自动生成缩略图，提供完整的Terraform配置和参数说明。
* [部署定时触发器](timer_trigger.md) - 通过定时触发器实现周期性任务执行，支持Cron表达式和固定间隔两种调度方式，提供完整的Terraform配置和参数说明。

## 参考资料

- [华为云FunctionGraph产品文档](https://support.huaweicloud.com/functiongraph/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
