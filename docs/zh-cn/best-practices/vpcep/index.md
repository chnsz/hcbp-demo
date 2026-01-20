# 简介

## 什么是VPC终端节点（VPCEP）

VPC终端节点（VPC Endpoint，VPCEP）是华为云提供的VPC内资源互访服务，支持在VPC内创建终端节点和终端节点服务，实现VPC内资源的私网访问。VPCEP服务支持跨VPC的私网访问，避免通过公网访问，提高访问安全性和稳定性，降低网络延迟和成本。

VPCEP服务提供完整的终端节点和终端节点服务生命周期管理功能，支持终端节点服务、终端节点等资源的创建和配置，具备高可靠性和高安全性。通过VPCEP服务，企业可以轻松实现跨VPC的私网访问，构建安全、高效的混合云架构。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云VPC终端节点（VPCEP）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的VPCEP资源。

通过本章节的最佳实践，您可以学习到主要的VPCEP资源的部署流程，这些最佳实践将帮助您快速上手VPCEP的自动化部署，并为后续的终端节点服务、终端节点管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署服务](service.md) - 介绍如何使用Terraform自动化部署终端节点服务，包括可用区查询、ECS规格查询、镜像查询、VPC、子网、安全组、ECS实例和终端节点服务的创建。

## 参考资料

- [华为云VPCEP产品文档](https://support.huaweicloud.com/vpcep/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
