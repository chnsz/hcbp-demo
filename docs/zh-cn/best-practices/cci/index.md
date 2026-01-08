# 简介

## 什么是云容器实例（CCI）

云容器实例（Cloud Container Instance，CCI）是华为云提供的Serverless容器服务，无需创建和管理服务器集群，即可直接运行容器应用。CCI提供秒级启动、按需付费、按秒计费的容器服务，支持Kubernetes原生API，让您专注于应用开发，无需关心基础设施的运维管理。

CCI服务支持多种容器镜像，提供灵活的资源配置和自动扩缩容能力，支持与VPC、ELB等服务的无缝集成。通过CCI，企业可以快速部署容器应用，享受Serverless容器的便利性和弹性，降低运维成本，提高开发效率。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云云容器实例（CCI）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的容器应用资源。

通过本章节的最佳实践，您可以学习到主要的CCI资源的部署流程，这些最佳实践将帮助您快速上手CCI的自动化部署，并为后续的容器应用管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署无状态负载](deployment.md) - 介绍如何使用Terraform自动化创建CCI无状态负载，包括命名空间的创建，实现容器的自动扩缩容、滚动更新和故障恢复等功能。
* [部署网络](network.md) - 介绍如何使用Terraform自动化创建CCI网络，包括VPC、子网、安全组和命名空间的创建，实现容器与云上其他资源的互通。

## 参考资料

- [华为云CCI产品文档](https://support.huaweicloud.com/cci/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
