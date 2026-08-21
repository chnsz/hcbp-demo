# 简介

## 什么是应用服务网格（ASM）

应用服务网格（Application Service Mesh，简称ASM）提供非侵入式的微服务治理解决方案，支持完整的生命周期管理和流量治理，兼容Kubernetes和Istio生态，功能包括负载均衡、熔断、故障注入等多种治理能力，并内置金丝雀、蓝绿灰度发布流程，提供一站式自动化的发布管理。

华为云ASM深度对接云容器引擎（CCE），以基础设施方式为用户提供服务流量管理、服务运行监控、服务访问安全以及服务发布能力，控制面和数据面均与开源Istio兼容，可为客户提供开箱即用的上手体验。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云应用服务网格（ASM）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的ASM资源。

通过本章节的最佳实践，您可以学习到主要的ASM资源的部署流程，这些最佳实践将帮助您快速上手ASM的自动化部署，并为后续的网格管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署基础网格](basic.md) - 介绍如何使用Terraform自动化部署一个关联单集群的基础ASM网格，包括指定网格名称、类型、版本，以及将网格组件安装到指定CCE集群节点。
* [部署多节点网格](multi_nodes.md) - 介绍如何使用Terraform自动化部署一个支持多节点安装与命名空间Sidecar注入的ASM网格，包括按需创建CCE命名空间、将网格组件安装到多个CCE节点，以及配置指定命名空间的Sidecar注入。
* [部署带命名空间的网格](with_namespace.md) - 介绍如何使用Terraform自动化部署一个配置命名空间Sidecar注入的ASM网格，包括按需创建CCE命名空间、将网格组件安装到指定CCE节点，以及配置指定命名空间的Sidecar注入。

## 参考资料

- [华为云ASM产品文档](https://support.huaweicloud.com/asm/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
