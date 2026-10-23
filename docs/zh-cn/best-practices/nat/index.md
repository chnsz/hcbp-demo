# 简介

## 什么是NAT网关（NAT Gateway）

NAT网关（NAT Gateway）是华为云提供的网络地址转换服务，能够为虚拟私有云（VPC）内的云资源提供访问公网以及被公网访问的能力，而无需为每台服务器单独绑定公网IP。NAT网关支持SNAT和DNAT两种转换方式：SNAT使VPC内的实例共享弹性公网IP访问互联网，DNAT则将公网EIP的端口映射到后端服务器的端口，使外部客户端能够访问VPC内的服务。

NAT网关提供从小型到超大型的多种规格，并可通过弹性公网IP灵活配置带宽，帮助企业在保障内部网络地址不暴露于公网的前提下，构建安全、可扩展且成本可控的出方向与入方向访问能力。

将NAT网关与VPC、子网、安全组和ECS实例结合使用，可以满足Web服务对外发布、远程登录、混合组网等多种公网访问场景，并通过基础设施即代码的方式实现统一管理。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云NAT网关（NAT Gateway）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的NAT网关资源。

通过本章节的最佳实践，您可以学习到主要的NAT网关资源的部署流程，这些最佳实践将帮助您快速上手NAT网关的自动化部署，并为后续的VPC、子网、安全组和ECS实例管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署DNAT规则](dnat_basic.md) - 介绍如何使用Terraform自动化部署DNAT规则，包括VPC与子网创建、NAT网关创建、EIP创建、后端ECS实例与安全组配置以及DNAT端口映射。

## 参考资料

- [华为云NAT网关产品文档](https://support.huaweicloud.com/natgateway/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
