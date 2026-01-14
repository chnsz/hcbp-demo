# 简介

## 什么是主机安全服务（HSS）

主机安全服务（Host Security Service，HSS）是华为云提供的主机安全防护服务，提供资产管理、漏洞管理、入侵检测、基线检查等功能，帮助您全面保护云上主机的安全。HSS服务支持多种操作系统，包括Linux和Windows，提供实时监控、威胁检测、安全加固等能力，满足企业级主机安全防护需求。

HSS服务提供主机分组管理功能，支持将多个主机进行分组管理，统一配置安全策略、执行安全检查和进行安全运维。通过HSS服务，企业可以实现主机安全的统一管理和监控，提高安全运维效率，保障业务系统的安全稳定运行。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云主机安全服务（HSS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的HSS资源。

通过本章节的最佳实践，您可以学习到主要的HSS资源的部署流程，这些最佳实践将帮助您快速上手HSS的自动化部署，并为后续的主机组管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署主机组](host_group.md) - 介绍如何使用Terraform自动化部署HSS主机组，包括VPC、子网、安全组、ECS实例（带HSS agent）和HSS主机组的创建。
* [部署主机防护](host_protection.md) - 介绍如何使用Terraform自动化部署HSS主机防护，为已存在的主机启用按需计费的HSS防护服务。

## 参考资料

- [华为云HSS产品文档](https://support.huaweicloud.com/hss/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
