# 简介

## 什么是虚拟专用网络（VPN）

虚拟专用网络（Virtual Private Network，VPN）是华为云提供的安全、可靠的网络连接服务，支持在VPC与本地网络之间建立加密的IPsec VPN连接，实现云上资源与本地数据中心的互联互通。VPN服务支持多种连接方式，包括站点到站点VPN、点到站点VPN等，满足不同场景的网络连接需求。

VPN服务提供完整的VPN网关生命周期管理功能，支持VPN网关、VPN连接、对端网关等资源的创建和配置，具备高可靠性和高安全性。通过VPN服务，企业可以轻松实现混合云架构，确保数据传输的安全性和稳定性，降低网络延迟和成本。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云虚拟专用网络（VPN）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的VPN资源。

通过本章节的最佳实践，您可以学习到主要的VPN资源的部署流程，这些最佳实践将帮助您快速上手VPN的自动化部署，并为后续的VPN网关、VPN连接管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署VPN网关](gateway.md) - 介绍如何使用Terraform自动化部署VPN网关，包括VPC、子网、EIP和VPN网关的创建。

## 参考资料

- [华为云VPN产品文档](https://support.huaweicloud.com/vpn/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
