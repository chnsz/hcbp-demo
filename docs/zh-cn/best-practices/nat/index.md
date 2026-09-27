# 简介

## 什么是NAT网关（NAT Gateway）

NAT网关（NAT Gateway）是华为云提供的高性能、高可用的公网地址转换服务，为企业提供安全、便捷的公网访问能力。NAT网关支持SNAT（源地址转换）和DNAT（目的地址转换）两种转换方式，能够帮助VPC内的云服务器在不绑定弹性公网IP的情况下访问公网或被公网访问，有效降低公网IP资源成本并提升网络安全性。

NAT网关提供多种规格，包括小型、中型、大型和超大型，满足不同规模和场景的业务需求。通过SNAT规则，VPC内的多台云服务器可以共享一个或多个弹性公网IP访问互联网；通过DNAT规则，可以将弹性公网IP的指定端口映射到VPC内云服务器的指定端口，实现对外提供服务。NAT网关还支持与VPC、子网、弹性公网IP等资源的灵活组合，帮助用户构建安全、高效的云上网络架构。

借助Terraform等基础设施即代码（IaC）工具，用户可以自动化地创建和管理NAT网关及其规则，实现网络资源的版本化管理和快速部署，为后续的网络运维和扩展奠定坚实基础。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云NAT网关（NAT Gateway）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的NAT网关资源。

通过本章节的最佳实践，您可以学习到主要的NAT网关资源的部署流程，这些最佳实践将帮助您快速上手NAT网关的自动化部署，并为后续的VPC、子网、弹性公网IP和云服务器管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署DNAT规则](dnat_basic.md) - 介绍如何使用Terraform自动化部署DNAT规则，包括VPC与子网创建、NAT网关创建、弹性公网IP创建、后端ECS实例创建和DNAT规则配置。

## 参考资料

- [华为云NAT网关产品文档](https://support.huaweicloud.com/natgateway/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
