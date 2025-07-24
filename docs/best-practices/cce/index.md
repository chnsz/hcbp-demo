# 云容器引擎（CCE）最佳实践

## 概述

云容器引擎（Cloud Container Engine，CCE）是一个高可靠高性能的企业级容器管理服务，支持Kubernetes社区原生应用和工具。它提供了多种类型的容器集群，方便用户部署、管理和扩展容器化应用。通过CCE，您可以轻松构建基于Kubernetes的容器化应用，并实现微服务架构。

CCE支持Standard和Turbo两种集群类型，提供全生命周期管理，支持多种类型节点包括ECS和BMS，提供弹性扩缩容能力。它支持VPC网络和容器隧道网络，实现容器间高效通信，并提供多种存储类型满足不同应用场景需求。

本章节提供了使用Terraform自动化部署和管理华为云云容器引擎（CCE）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的容器资源。

## 最佳实践列表

本章节包含以下最佳实践：

* [使用Terraform部署按需计费CCE集群](postpaid_cluster.md) - 介绍如何使用Terraform自动化部署按需计费的CCE集群，包括VPC、子网、CCE集群和节点的创建。

## 参考资料

- [华为云CCE产品文档](https://support.huaweicloud.com/cce/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
