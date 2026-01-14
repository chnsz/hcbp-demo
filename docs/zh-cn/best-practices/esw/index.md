# 简介

## 什么是企业交换机（ESW）

企业交换机（Enterprise Switch，ESW）是华为云提供的高性能、高可用的企业级网络交换服务，支持大二层网络互通，实现跨可用区的二层网络连接。ESW服务提供虚拟化网络交换能力，支持VPC内和跨VPC的二层网络互通，满足企业级网络组网需求。

ESW服务支持高可用部署模式，提供主备可用区部署，确保网络服务的高可用性。通过ESW服务，企业可以实现跨可用区的二层网络互通，支持虚拟机迁移、负载均衡、高可用部署等场景，满足企业级网络架构需求。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云企业交换机（ESW）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的ESW资源。

通过本章节的最佳实践，您可以学习到主要的ESW资源的部署流程，这些最佳实践将帮助您快速上手ESW的自动化部署，并为后续的ESW管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署实例](instance.md) - 介绍如何使用Terraform自动化部署ESW实例，包括VPC、子网和ESW实例的创建。
* [部署连接](connection.md) - 介绍如何使用Terraform自动化部署ESW连接，包括VPC、子网、ESW实例和ESW连接的创建。
* [部署连接与虚拟端口绑定](connection_vport_bind.md) - 介绍如何使用Terraform自动化部署ESW连接与虚拟端口绑定，包括VPC、子网、ESW实例、ESW连接、VPC子网私有IP和连接虚拟端口绑定的创建。

## 参考资料

- [华为云ESW产品文档](https://support.huaweicloud.com/esw/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
