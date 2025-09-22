# 简介

## 什么是云硬盘（EVS）

云硬盘（Elastic Volume Service，EVS）是华为云提供的高性能、高可靠、可扩展的块存储服务，为ECS实例提供持久化存储。EVS支持多种存储类型，包括SSD、SAS、SATA等，满足不同业务场景的存储需求。

通过EVS，您可以创建、挂载、卸载和管理云硬盘，为ECS实例提供持久化存储能力。华为云EVS支持快照功能，可以创建数据备份，支持快照组功能，实现多个云硬盘的一致性备份。华为云EVS支持与ECS、VPC等服务的无缝集成，为您提供完整的存储解决方案。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云云硬盘（EVS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的存储资源。

通过本章节的最佳实践，您可以学习到主要的云硬盘资源的部署流程，这些最佳实践将帮助您快速上手云硬盘的自动化部署，并为后续的EVS管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署磁盘快照](snapshot.md) - 介绍如何使用Terraform自动化部署EVS磁盘快照，包括云硬盘创建和快照配置。
* [部署磁盘快照组](snapshot_group.md) - 介绍如何使用Terraform自动化部署EVS磁盘快照组，包括ECS实例创建、云硬盘创建、挂载和磁盘快照组配置。

## 参考资料

- [华为云EVS产品文档](https://support.huaweicloud.com/evs/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
