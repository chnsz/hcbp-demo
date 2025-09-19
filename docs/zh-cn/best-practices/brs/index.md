# 简介

## 什么是业务容灾服务（BRS）

业务恢复服务（Business Recovery Service，BRS）是一种为弹性云服务器（Elastic Cloud Server，ECS）和云硬盘（Elastic Volume Service，EVS）等服务提供容灾的服务。通过主机层复制、数据冗余和缓存加速等多项技术，提供给用户高级别的数据可靠性以及业务连续性，称为业务恢复服务。

业务恢复服务有助于保护业务应用，将弹性云服务器的数据、配置信息复制到容灾站点，并允许业务应用在生产站点云服务器停机期间在容灾站点云服务器上启动并正常运行，从而提升业务连续性。BRS支持多种容灾模式，包括同城容灾、异地容灾等，并提供灾难恢复演练功能，让您能够定期验证容灾方案的可行性。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云业务容灾服务（BRS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的容灾资源。

通过本章节的最佳实践，您可以学习到主要的容灾资源的部署流程，这些最佳实践将帮助您快速上手业务容灾服务的自动化部署，并为后续的保护组、受保护实例和灾难恢复演练管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署灾难恢复演练](disaster_recovery_drill.md) - 介绍如何使用Terraform自动化部署BRS灾难恢复演练环境，包括保护组、受保护实例和灾难恢复演练的创建。
* [部署保护组](protection_group.md) - 介绍如何使用Terraform自动化部署BRS保护组环境，包括保护组的创建。
* [部署保护实例](protected_instance.md) - 介绍如何使用Terraform自动化部署BRS保护实例环境，包括保护组和受保护实例的创建。

## 参考资料

- [华为云BRS产品文档](https://support.huaweicloud.com/sdrs/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
