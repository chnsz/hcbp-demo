# 简介

## 什么是Anti-DDoS（Anti-DDoS）

Anti-DDoS（Anti-Distributed Denial of Service）是华为云提供的分布式拒绝服务攻击防护服务，能够有效防护针对公网IP的DDoS攻击，保障业务的稳定运行。Anti-DDoS服务提供基础防护和专业版防护两种防护模式，基础防护为华为云用户提供免费的DDoS攻击防护能力，当检测到DDoS攻击时，系统会自动启动流量清洗，将攻击流量过滤后，仅将正常流量转发给源站服务器。

Anti-DDoS服务支持多种攻击类型的防护，包括SYN Flood、UDP Flood、ICMP Flood、HTTP Flood等常见DDoS攻击类型。通过配置流量清洗阈值和消息通知，用户可以及时了解防护状态，确保业务安全稳定运行。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云Anti-DDoS的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的Anti-DDoS防护资源。

通过本章节的最佳实践，您可以学习到主要的Anti-DDoS防护资源的部署流程，这些最佳实践将帮助您快速上手Anti-DDoS的自动化部署，并为后续的Anti-DDoS管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署Anti-DDoS基础防护](basic.md) - 介绍如何使用Terraform自动化部署Anti-DDoS基础防护，包括创建弹性公网IP、消息通知服务主题和订阅，以及配置Anti-DDoS基础防护。
* [部署Anti-DDoS默认防护策略](default_protection_policy.md) - 介绍如何使用Terraform自动化部署Anti-DDoS默认防护策略，为账户下所有EIP提供统一的防护配置。

## 参考资料

- [华为云Anti-DDoS产品文档](https://support.huaweicloud.com/antiddos/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
