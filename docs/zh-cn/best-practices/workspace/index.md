# 云桌面（Workspace）最佳实践

## 概述

华为云云桌面（Workspace）是一种基于云计算的桌面虚拟化服务，为企业用户提供安全、便捷的云上办公解决方案。它支持Windows和Linux等多种操作系统，提供灵活的计费模式和丰富的终端接入方式，帮助企业快速构建安全、高效的远程办公环境。

通过云桌面，您可以实现数据集中存储、终端轻量化，用户可以随时随地通过各种终端设备安全地访问云上的办公桌面。华为云云桌面支持与VPC、安全组、云硬盘等服务的无缝集成，为您提供完整的云上办公解决方案。

本章节提供了使用Terraform自动化部署和管理华为云云桌面（Workspace）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的桌面虚拟化资源。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署按需计费的云桌面](postpaid_desktop.md) - 介绍如何使用Terraform自动化部署按需计费的云桌面实例，包括VPC、子网、安全组以及云桌面服务和用户的创建。

## 参考资料

- [华为云云桌面产品文档](https://support.huaweicloud.com/workspace/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
