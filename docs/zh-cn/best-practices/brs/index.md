# 业务容灾服务（BRS）最佳实践

## 概述

业务恢复服务（Business Recovery Service，BRS）是一种为弹性云服务器（Elastic Cloud Server，ECS）和云硬盘（Elastic Volume Service，EVS）等服务提供容灾的服务。通过主机层复制、数据冗余和缓存加速等多项技术，提供给用户高级别的数据可靠性以及业务连续性，称为业务恢复服务。

业务恢复服务有助于保护业务应用，将弹性云服务器的数据、配置信息复制到容灾站点，并允许业务应用在生产站点云服务器停机期间在容灾站点云服务器上启动并正常运行，从而提升业务连续性。BRS支持多种容灾模式，包括同城容灾、异地容灾等，并提供灾难恢复演练功能，让您能够定期验证容灾方案的可行性。通过Terraform自动化部署，您可以快速构建完整的容灾环境，实现基础设施即代码的容灾管理。

本章节提供了使用Terraform自动化部署和管理华为云业务容灾服务（BRS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的容灾资源。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署灾难恢复演练](disaster_recovery_drill.md) - 介绍如何使用Terraform自动化部署BRS灾难恢复演练环境，包括保护组、受保护实例和灾难恢复演练的创建。

## 参考资料

- [华为云BRS产品文档](https://support.huaweicloud.com/sdrs/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
