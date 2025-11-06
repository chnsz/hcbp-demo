# 简介

## 什么是虚拟私有云（VPC）

虚拟私有云（Virtual Private Cloud，VPC）是华为云为用户提供的逻辑隔离网络环境，支持自定义子网、路由、ACL等网络资源，满足企业级网络安全和灵活组网需求。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云VPC的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的网络资源。

通过本章节的最佳实践，您可以学习到主要的网络资源的部署流程，这些最佳实践将帮助您快速上手虚拟私有云的自动化部署，并为后续的VPC管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署基础网络](basic.md) - 介绍如何使用Terraform自动化部署基础VPC及子网。
* [部署对等连接](peering.md) - 介绍如何使用Terraform自动化部署两个VPC及其子网，并建立VPC Peering连接和路由。
* [部署安全组](security_group.md) - 介绍如何使用Terraform自动化部署安全组及其规则配置。
* [部署VIP并关联实例](vip_associate.md) - 介绍如何使用Terraform自动化部署VIP关联，实现高可用性部署。

## 参考资料

- [华为云VPC产品文档](https://support.huaweicloud.com/vpc/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
