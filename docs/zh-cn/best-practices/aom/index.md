# 简介

## 什么是应用运维管理（AOM）

应用运维管理（Application Operations Management，AOM）是华为云提供的一站式应用运维管理平台，为企业提供统一的应用运维管理入口。AOM通过应用监控、日志管理、告警管理等功能，帮助企业实现自动化运维和智能化管理，提升运维效率和质量。

AOM服务支持多种监控指标和告警规则，提供完整的应用生命周期管理能力，支持与云日志服务、消息通知服务等服务的无缝集成，帮助企业构建完善的运维管理体系。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云AOM的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的应用运维管理资源。

通过本章节的最佳实践，您可以学习到主要的AOM资源的部署流程，这些最佳实践将帮助您快速上手AOM的自动化部署，并为后续的AOM管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署AOM告警动作回调](action_callback.md) - 介绍如何使用Terraform自动化部署AOM告警动作回调，包括创建SMN主题和订阅、AOM消息模板，以及配置AOM告警动作规则。

## 参考资料

- [华为云AOM产品文档](https://support.huaweicloud.com/aom/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
