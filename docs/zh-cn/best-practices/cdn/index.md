# 简介

## 什么是内容分发网络（CDN）

内容分发网络（Content Delivery Network，CDN）是华为云提供的内容加速分发服务，通过将内容缓存到全球分布的边缘节点，为用户提供就近访问的加速体验。CDN服务支持静态内容加速、动态内容加速、视频点播和直播加速等多种场景，帮助用户提升网站访问速度，降低源站压力，提升用户体验。

CDN服务提供灵活的缓存策略配置、HTTPS加速、防盗链、访问控制等安全功能，支持实时监控和统计分析，具备高可用性和高可靠性。通过CDN服务，企业可以快速构建全球内容分发网络，实现内容的快速分发和访问，降低带宽成本，提升业务竞争力。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云内容分发网络（CDN）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的CDN资源。

通过本章节的最佳实践，您可以学习到主要的CDN资源的部署流程，这些最佳实践将帮助您快速上手CDN的自动化部署，并为后续的CDN管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署缓存管理](cache_management.md) - 介绍如何使用Terraform自动化执行CDN缓存刷新和预热操作，管理CDN节点上的缓存内容，确保用户获取到最新的资源并提高访问速度。
* [部署HTTPS和缓存域名](domain_with_https_and_cache.md) - 介绍如何使用Terraform自动化创建CDN域名，包括HTTPS和缓存规则的配置，实现内容加速和安全传输。

## 参考资料

- [华为云CDN产品文档](https://support.huaweicloud.com/cdn/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
