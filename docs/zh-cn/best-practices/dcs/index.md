# 简介

## 什么是分布式缓存服务（DCS）

分布式缓存服务（Distributed Cache Service, DCS）是华为云提供的高性能、高可用的内存数据库服务，支持Redis、Memcached等主流缓存引擎。DCS服务提供多种实例规格和部署模式，包括单机、主备、集群等，满足不同规模和场景的缓存需求。

DCS服务提供完整的缓存生命周期管理功能，支持自动备份、监控告警、参数调优等企业级功能，具备高可靠性和高安全性。通过DCS服务，企业可以轻松实现数据缓存、会话存储、消息队列等应用场景，提升应用性能和用户体验。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云分布式缓存服务（DCS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的缓存资源。

通过本章节的最佳实践，您可以学习到主要的DCS资源的部署流程，这些最佳实践将帮助您快速上手DCS的自动化部署，并为后续的缓存管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署主备Redis实例](redis_ha_instance.md) - 介绍如何使用Terraform自动化部署DCS主备Redis实例，包括VPC创建、实例配置、备份策略和白名单管理。
* [部署Redis账号管理](redis_account.md) - 介绍如何使用Terraform自动化部署DCS Redis实例及账号，包括VPC创建、子网、实例配置和账号权限管理。
* [部署Redis后台任务删除](redis_background_task_delete.md) - 介绍如何使用Terraform删除DCS Redis实例中指定的后台任务，适用于清理已完成或异常的后台任务。
* [部署Redis大Key分析](redis_bigkey_analysis.md) - 介绍如何使用Terraform自动化部署DCS Redis实例并创建大Key分析任务，包括VPC创建、子网、实例配置和大Key分析。
* [部署Redis中心任务删除](redis_center_task_delete.md) - 介绍如何使用Terraform自动化部署Redis中心任务删除，包括DCS中心任务删除（huaweicloud_dcs_center_task_delete）。
* [部署Redis自定义模板](redis_custom_template.md) - 介绍如何使用Terraform自动化部署Redis自定义模板，包括DCS自定义模板（huaweicloud_dcs_custom_template）。
* [部署Redis实例全会话清理](redis_all_sessions_kill.md) - 介绍如何使用Terraform自动化部署DCS Redis实例并执行全会话清理，包括VPC创建、实例配置和会话清理操作。
* [部署Redis实例备份](redis_backup.md) - 介绍如何使用Terraform自动化部署DCS Redis实例并创建手动备份，包括VPC创建、实例配置和备份资源管理。
* [部署单机Redis实例](redis_single_instance.md) - 介绍如何使用Terraform自动化部署DCS单机Redis实例，包括VPC创建、实例配置和基础网络设置。

## 参考资料

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
