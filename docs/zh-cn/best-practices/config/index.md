# 简介

## 什么是配置审计（Config）

配置审计（Config）是华为云提供的一站式合规管理服务，帮助用户持续监控和评估云资源的配置合规性。Config服务提供预置的合规规则包和自定义规则，支持多种合规框架和标准，帮助企业建立完善的合规管理体系。

通过Config，您可以监控云资源的配置变更，评估资源配置是否符合安全、合规要求，及时发现和修复配置风险。Config服务支持多种合规框架，包括CIS、NIST、SOC等国际标准，以及华为云安全最佳实践，为企业提供全面的合规管理解决方案。华为云Config支持与云监控、消息通知等服务的无缝集成，为您提供完整的合规监控和告警体系。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云配置审计（Config）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的合规资源。

通过本章节的最佳实践，您可以学习到主要的配置审计资源的部署流程，这些最佳实践将帮助您快速上手配置审计的自动化部署，并为后续的Config管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署合规规则包](compliance_package.md) - 介绍如何使用Terraform自动化部署Config合规规则包，包括规则包模板查询和合规规则包创建。
* [部署合规规则](compliance_rule.md) - 介绍如何使用Terraform自动化部署Config合规规则，包括VPC创建、ECS实例创建、合规规则配置和规则评估。

## 参考资料

- [华为云Config产品文档](https://support.huaweicloud.com/rms/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
