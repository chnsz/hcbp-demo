# 简介

## 什么是数据管理服务（DAS）

数据管理服务（Data Admin Service，简称DAS）是用来登录和操作华为云上数据库的Web服务。它提供数据库开发、运维、智能诊断的一站式云上数据库管理平台，方便用户使用和运维数据库。

DAS提供数据库可视化操作能力，包括基础SQL操作、高级数据库管理和智能化运维等功能，旨在帮助用户易用、安全、智能地进行数据库管理。无需安装本地客户端，即可对RDS、GaussDB、DDS等多种数据库实例进行可视化管理与运维。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云数据管理服务（DAS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的DAS资源。

通过本章节的最佳实践，您可以学习到主要的DAS资源的部署流程，这些最佳实践将帮助您快速上手DAS的自动化部署，并为后续的数据库连接、锁分析、数据库用户和共享连接管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署数据库连接](database_connection.md) - 介绍如何使用Terraform自动化部署DAS数据库实例连接，包括创建数据库实例连接、数据库用户，以及将连接共享给其他IAM用户。
* [部署锁分析](lock_analysis.md) - 介绍如何使用Terraform自动化部署DAS锁分析相关配置，包括开启全量死锁检测、开启历史事务开关，以及创建历史事务导出任务。

## 参考资料

- [华为云DAS产品文档](https://support.huaweicloud.com/das/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
