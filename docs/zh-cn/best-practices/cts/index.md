# 云审计服务（CTS）最佳实践

## 概述

云审计服务（Cloud Trace Service，CTS）是华为云提供的一项安全服务，用于记录用户在云服务账户中进行的所有操作，包括通过控制台、API、SDK等方式进行的操作。CTS提供操作记录查询、审计和问题定位功能，支持将操作记录保存到OBS桶中，满足安全审计、合规性检查等需求。

作为华为云的核心安全服务之一，CTS支持多种云服务的操作记录，包括ECS、VPC、OBS、RDS等，能够帮助您实现完整的云上操作审计。CTS提供实时监控、历史查询、告警通知等功能，让您能够全面了解云资源的使用情况和操作历史。

本章节提供了使用Terraform自动化部署和管理华为云云审计服务（CTS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的审计和监控资源。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署数据跟踪器](data_tracker.md) - 通过数据跟踪器实现云资源操作的自动记录和存储，支持安全审计、合规监控等功能，提供完整的Terraform配置和参数说明。
* [部署通知](notification.md) - 通过CTS通知实现云资源操作的实时监控和告警，支持安全事件监控、异常操作告警等功能，提供完整的Terraform配置和参数说明。
* [部署系统追踪器](system_tracker.md) - 通过系统追踪器实现云资源操作的自动记录和存储，支持安全审计、合规监控等功能，提供完整的Terraform配置和参数说明。

## 参考资料

- [华为云CTS产品文档](https://support.huaweicloud.com/cts/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
