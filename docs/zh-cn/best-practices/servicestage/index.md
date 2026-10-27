# 简介

## 什么是应用管理与运维平台（ServiceStage）

应用管理与运维平台（ServiceStage）是华为云提供的一站式应用托管平台，面向企业应用与云原生应用，提供应用开发、构建、部署、运维的全生命周期管理能力。ServiceStage支持虚拟机、容器、Serverless等多种部署形态，帮助企业在云上快速构建并运行应用，降低应用上云与运维的复杂度。

ServiceStage提供应用编排、环境管理、配置管理、微服务治理、灰度发布、弹性伸缩等核心能力。通过配置组与配置文件，用户可以将应用配置集中管理并在不同环境之间复用；通过微服务引擎与治理能力，用户可以构建高可用、可观测的分布式应用系统。

借助ServiceStage，企业可以将应用交付流程标准化，实现从代码到运行环境的一致交付，并为后续的应用运维、监控与治理工作奠定基础。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云应用管理与运维平台（ServiceStage）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的ServiceStage资源。

通过本章节的最佳实践，您可以学习到主要的ServiceStage资源的部署流程，这些最佳实践将帮助您快速上手ServiceStage的自动化部署，并为后续的应用配置、应用部署和应用运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署配置组与配置文件](configuration_group_with_configuration.md) - 介绍如何使用Terraform自动化部署配置组与配置文件，包括配置组创建、配置文件创建和配置文件内容设置。
* [部署环境与应用及其应用配置](environment_with_application.md) - 介绍如何使用Terraform自动化部署环境与应用及其应用配置，包括VPC创建、环境创建、应用创建和环境变量配置。

## 参考资料

- [华为云ServiceStage产品文档](https://support.huaweicloud.com/servicestage/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
