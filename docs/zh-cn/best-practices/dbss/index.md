# 简介

## 什么是数据库安全服务（DBSS）

数据库安全服务（Database Security Service，DBSS）提供数据库安全审计、数据库安全加密、数据库安全运维功能。数据库审计通过实时记录用户访问数据库行为，形成细粒度的审计报告，对风险行为和攻击行为进行实时告警。

数据库安全审计采用旁路部署方式，支持对华为云上的RDS、ECS/BMS自建数据库进行审计，在不影响用户业务的前提下灵活开展审计，帮助企业对内部违规和不正当操作进行定位追责，保障数据资产安全。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云数据库安全服务（DBSS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的DBSS资源。

通过本章节的最佳实践，您可以学习到主要的DBSS资源的部署流程，这些最佳实践将帮助您快速上手DBSS的自动化部署，并为后续的数据库安全审计实例和ECS自建数据库审计管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署审计ECS数据库](audit_ecs_database.md) - 介绍如何使用Terraform自动化部署DBSS审计实例并添加ECS自建数据库，包括创建VPC、子网、安全组、DBSS实例，以及配置自建数据库审计信息。

## 参考资料

- [华为云DBSS产品文档](https://support.huaweicloud.com/dbss/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
