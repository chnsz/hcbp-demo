# 简介

## 什么是应用管理与运维平台（ServiceStage）

应用管理与运维平台（ServiceStage）是华为云提供的一站式应用管理服务，面向企业应用与云原生应用，提供应用开发、构建、部署、治理与运维的全生命周期管理能力。ServiceStage帮助用户屏蔽底层基础设施差异，实现应用的快速上云与统一管理，降低应用交付与运维的复杂度。

ServiceStage提供应用管理、组件管理、环境管理、配置管理、微服务治理等核心能力，支持虚拟机、容器、Serverless等多种部署形态，兼容Spring Cloud、Dubbo等主流微服务框架，并可与CSE、AOM、LTS等云服务协同，构建完整的应用运行与治理体系。

通过ServiceStage，用户可以集中管理应用的环境变量、启动参数与业务配置，实现配置与代码分离，并借助灰度发布、全链路监控等能力提升应用的可用性与运维效率，为业务的持续交付与稳定运行提供支撑。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云应用管理与运维平台（ServiceStage）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的ServiceStage资源。

通过本章节的最佳实践，您可以学习到主要的ServiceStage资源的部署流程，这些最佳实践将帮助您快速上手ServiceStage的自动化部署，并为后续的应用、组件、配置管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署配置组与配置文件](configuration_group_with_configuration.md) - 介绍如何使用Terraform自动化部署配置组与配置文件，包括配置组创建、配置文件创建与内容配置。

## 参考资料

- [华为云ServiceStage产品文档](https://support.huaweicloud.com/servicestage/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
