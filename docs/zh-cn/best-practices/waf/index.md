# Web应用防火墙（WAF）最佳实践

## 概述

Web应用防火墙（Web Application Firewall，WAF）是华为云为Web应用提供的一站式安全防护服务，可有效防御SQL注入、XSS跨站脚本、网页木马上传、命令注入、恶意爬虫、CC攻击等多种Web安全威胁。支持云模式和独享实例（专业版），满足企业级高性能和合规需求。

通过WAF，您可以为网站和Web应用构建多层次的安全防线，确保业务的可用性、安全性和合规性。华为云WAF支持与CDN、ELB、API网关等服务的无缝集成，为您提供完整的Web安全解决方案。

本章节提供了使用Terraform自动化部署和管理华为云Web应用防火墙（WAF）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的Web安全资源。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署WAF专业版Domain](dedicated_domain.md) - 介绍如何使用Terraform自动化部署WAF专业版Domain，包括WAF专业版实例、WAF策略和域名配置的创建。
* [部署WAF专业版实例](dedicated_instance.md) - 介绍如何使用Terraform自动化部署WAF专业版实例，包括VPC、子网和安全组的创建。

## 参考资料

- [华为云WAF产品文档](https://support.huaweicloud.com/waf/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
