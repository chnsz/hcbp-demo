# 简介

## 什么是资源访问管理（RAM）

资源访问管理（Resource Access Manager，RAM）是华为云提供的资源共享服务，支持跨账号的资源共享和管理，帮助您实现资源的统一管理和访问控制。RAM服务提供集中化的资源共享能力，支持跨账号的资源分享、权限管理和访问控制，满足企业级资源共享和合规要求。

RAM服务支持多种资源类型的共享，包括VPC、子网、安全组、路由表等网络资源，以及ECS、RDS等计算和存储资源。通过RAM服务，企业可以实现跨账号的资源共享，提高资源利用率，降低管理成本，同时保障资源的安全性和合规性。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云资源访问管理（RAM）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的RAM资源。

通过本章节的最佳实践，您可以学习到主要的RAM资源的部署流程，这些最佳实践将帮助您快速上手RAM的自动化部署，并为后续的资源分享、权限管理和访问控制运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署自动化资源分享邀请处理](automated_resource_share_invitation_processing.md) - 介绍如何使用Terraform自动化处理资源分享邀请，包括查询待处理的邀请和批量接受或拒绝邀请。
* [部署跨账号资源分享](cross_account_resource_share.md) - 介绍如何使用Terraform自动化部署跨账号资源分享，包括资源分享实例的创建、主体配置、资源URN配置和权限配置。

## 参考资料

- [华为云RAM产品文档](https://support.huaweicloud.com/ram/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
