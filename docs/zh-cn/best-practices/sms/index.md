# 简介

## 什么是主机迁移服务（SMS）

主机迁移服务（Server Migration Service，SMS）是华为云提供的服务器迁移服务，支持将物理服务器、虚拟机或其他云平台的服务器迁移到华为云，实现业务的无缝迁移。SMS服务提供完整的迁移解决方案，支持Windows和Linux操作系统的迁移，支持在线迁移和离线迁移两种方式，满足不同场景的迁移需求。

SMS服务支持多种源端环境，包括物理服务器、VMware、Hyper-V、KVM、XenServer等虚拟化平台，以及AWS、Azure、阿里云等主流云平台。通过SMS服务，您可以实现服务器的批量迁移、增量同步和自动化部署，降低迁移成本和风险，提高迁移效率。SMS服务还支持迁移过程中的数据一致性保障和业务连续性保障，确保迁移过程的安全可靠。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云主机迁移服务（SMS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的SMS资源。

通过本章节的最佳实践，您可以学习到主要的SMS资源的部署流程，这些最佳实践将帮助您快速上手SMS的自动化部署，并为后续的迁移项目管理、迁移任务管理和迁移监控运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署迁移项目](migration_project.md) - 介绍如何使用Terraform自动化部署迁移项目，包括项目基本信息、区域配置、网络配置和同步策略配置。
* [部署迁移任务](migration_task.md) - 介绍如何使用Terraform自动化部署迁移任务，包括查询可用区和源服务器、创建服务器模板和创建迁移任务。

## 参考资料

- [华为云SMS产品文档](https://support.huaweicloud.com/sms/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
