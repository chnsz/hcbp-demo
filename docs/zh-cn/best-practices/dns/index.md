# 简介

## 什么是云解析服务（DNS）

云解析服务（Domain Name Service, DNS）是华为云提供的高可用、高性能的域名解析服务，支持公网域名解析和私网域名解析。DNS服务提供智能解析、负载均衡、健康检查等功能，帮助用户实现域名的智能调度和故障转移。

DNS服务支持多种解析类型，包括A记录、AAAA记录、CNAME记录、MX记录等，满足不同场景的域名解析需求。通过DNS服务，企业可以实现域名的高可用解析、智能调度、故障转移等功能，提升应用的可用性和用户体验。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云云解析服务（DNS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的域名解析资源。

通过本章节的最佳实践，您可以学习到主要的DNS资源的部署流程，这些最佳实践将帮助您快速上手DNS的自动化部署，并为后续的域名解析管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署自定义线路](custom_line.md) - 介绍如何使用Terraform自动化部署DNS自定义线路，包括线路创建和IP地址段配置。
* [部署终端节点](endpoint.md) - 介绍如何使用Terraform自动化部署DNS终端节点，包括VPC创建、子网配置和终端节点部署。

## 参考资料

- [华为云DNS产品文档](https://support.huaweicloud.com/dns/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
