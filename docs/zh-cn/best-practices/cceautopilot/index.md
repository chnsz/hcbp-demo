# 简介

## 什么是云容器引擎（CCE）Autopilot

云容器引擎（Cloud Container Engine，CCE）Autopilot是华为云提供的Serverless化Kubernetes集群形态，面向云原生应用提供免运维的容器运行环境。Autopilot集群将集群节点的管理与运维工作交由华为云负责，用户无需创建、配置和扩缩节点，只需关注业务负载本身，即可获得高可靠、高弹性的容器运行能力。

Autopilot集群采用ENI容器网络模式，支持按需弹性伸缩与按量计费，能够根据业务负载自动调整底层资源，帮助用户在保障业务稳定性的同时有效控制成本。集群兼容Kubernetes原生API与生态工具，支持插件化扩展，可通过安装日志采集、监控等插件满足可观测性与运维需求。

通过CCE Autopilot，企业可以快速构建面向微服务、在线业务、批处理等场景的容器平台，降低集群运维复杂度，提升资源利用率与业务交付效率。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云云容器引擎（CCE）Autopilot的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的CCE Autopilot资源。

通过本章节的最佳实践，您可以学习到主要的CCE Autopilot资源的部署流程，这些最佳实践将帮助您快速上手CCE Autopilot的自动化部署，并为后续的集群、插件管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署CCE Autopilot集群插件](cce_autopilot_addons.md) - 介绍如何使用Terraform自动化部署CCE Autopilot集群插件，包括VPC创建、子网创建、集群创建、SWR组织创建和log-agent插件安装。

## 参考资料

- [华为云云容器引擎产品文档](https://support.huaweicloud.com/cce/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
