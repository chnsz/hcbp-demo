# 简介

## 什么是云运维中心（COC）

云运维中心（Cloud Operations Center, COC）是华为云提供的一站式运维管理平台，为企业提供统一的运维管理入口。云运维中心通过脚本管理、任务调度、监控告警等功能，帮助企业实现自动化运维和智能化管理，提升运维效率和质量。

云运维中心提供完整的运维生命周期管理功能，支持多种脚本类型和任务执行方式，具备高可靠性和高安全性。通过云运维中心，企业可以轻松实现运维自动化，确保业务稳定运行，降低运维成本和风险。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云云运维中心（COC）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的运维资源。

通过本章节的最佳实践，您可以学习到主要的COC资源的部署流程，这些最佳实践将帮助您快速上手COC的自动化部署，并为后续的脚本管理和任务调度工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署脚本](script.md) - 介绍如何使用Terraform自动化部署COC脚本，包括脚本创建和参数配置。
* [部署脚本执行](script_execution.md) - 介绍如何使用Terraform自动化部署COC脚本执行，包括ECS实例、脚本创建和脚本执行的完整流程。
* [部署脚本订单执行](script_order_execution.md) - 介绍如何使用Terraform自动化部署COC脚本订单执行，包括ECS实例、脚本创建、脚本执行和订单操作的完整流程。

## 参考资料

- [华为云COC产品文档](https://support.huaweicloud.com/coc/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
