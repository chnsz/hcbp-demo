# 事件网格（EG）最佳实践

## 概述

事件网格（EventGrid，EG）是华为云提供的事件驱动架构服务，支持事件的生产、路由、转换和消费。通过事件网格，您可以构建松耦合、可扩展的分布式应用系统，实现事件驱动的微服务架构。

事件网格支持多种事件源和目标，包括云服务事件、自定义事件、第三方系统事件等，提供灵活的事件过滤、转换和路由能力。它支持事件订阅、事件通道管理、事件源配置等功能，帮助企业实现高效的事件驱动架构。

作为华为云的核心集成服务之一，事件网格支持与FunctionGraph、OBS、SMN、APIG等服务的无缝集成，能够满足复杂的事件驱动场景需求。事件网格提供高可靠性、低延迟的事件传递能力，支持事件重试、死信队列等高级功能。

本章节提供了使用Terraform自动化部署和管理华为云事件网格（EG）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的事件驱动资源。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署事件订阅（自定义事件源，EG事件目标）](event_subscription_custom_to_eg.md) - 介绍如何使用Terraform自动化部署事件订阅，通过自定义事件源和EG事件目标实现事件订阅和路由，支持事件过滤、转换和分发等功能。
* [部署事件订阅（OBS事件源、Kafka事件目标）](event_subscription_obs_to_kafka.md) - 介绍如何使用Terraform自动化部署事件订阅，通过OBS事件源和Kafka事件目标实现事件订阅和路由，支持OBS存储操作事件的实时推送和Kafka消息队列集成。

## 参考资料

- [华为云事件网格产品文档](https://support.huaweicloud.com/eg/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
