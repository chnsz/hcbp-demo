# 简介

## 什么是云容器引擎（CCE）Autopilot

云容器引擎（Cloud Container Engine，CCE）Autopilot是华为云提供的Serverless化Kubernetes集群形态，面向云原生应用提供免运维的容器编排能力。Autopilot集群将控制面和节点资源完全托管，用户无需创建和管理节点，只需提交工作负载即可运行容器化应用，从而专注于业务开发与创新。

CCE Autopilot集群采用ENI容器网络模式，支持Kubernetes社区原生API与工具，兼容主流云原生生态。集群内置弹性伸缩、负载均衡、日志与监控等能力，并支持通过插件市场按需安装日志采集、监控、网络等插件，帮助用户快速构建可观测、可扩展的容器化应用平台。

通过CCE Autopilot，企业可以按实际使用的计算资源付费，降低集群运维成本与资源闲置，同时借助插件化能力完善日志、监控、审计等运维体系，为后续的容器化业务管理和运维工作奠定坚实基础。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云云容器引擎（CCE）Autopilot的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的CCE Autopilot资源。

通过本章节的最佳实践，您可以学习到主要的CCE Autopilot资源的部署流程，这些最佳实践将帮助您快速上手CCE Autopilot的自动化部署，并为后续的集群、插件、网络管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署CCE Autopilot集群插件](cce_autopilot_addons.md) - 介绍如何使用Terraform自动化部署CCE Autopilot集群插件，包括VPC和子网创建、CCE Autopilot集群创建、SWR组织创建和log-agent插件部署。
* [部署CCE Autopilot应用](cce_autopilot_chart_release.md) - 介绍如何使用Terraform自动化部署CCE Autopilot应用，包括VPC和子网创建、CCE Autopilot集群创建、Helm Chart上传和Helm Release部署。

## 参考资料

- [华为云云容器引擎产品文档](https://support.huaweicloud.com/cce/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
