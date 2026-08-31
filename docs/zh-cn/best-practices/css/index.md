# 简介

## 什么是云搜索服务（CSS）

云搜索服务（Cloud Search Service，简称CSS）是华为云ELK生态的一系列软件集合，是一个基于Elasticsearch、OpenSearch且完全托管的在线分布式搜索服务，为用户提供更丰富、更灵活的检索和分析功能，可处理结构化、非结构化文本及基于AI向量的多条件检索、统计和报表等功能。

CSS服务兼容Elasticsearch、OpenSearch、Logstash、Kibana、OpenSearch Dashboards、Cerebro等软件，支持自动部署，可快速创建集群，拥有免运维、完善的监控体系等特点，适用于日志分析、智能客服、知识库问答、个性化推荐等多种业务场景。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云云搜索服务（CSS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的CSS资源。

通过本章节的最佳实践，您可以学习到主要的CSS资源的部署流程，这些最佳实践将帮助您快速上手CSS的自动化部署，并为后续的搜索集群管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署AI Ops设置](ai_ops_setting.md) - 介绍如何使用Terraform自动化部署CSS AI Ops设置，包括创建VPC、子网、安全组、CSS集群，以及配置AI Ops巡检设置。
* [部署集群](cluster.md) - 介绍如何使用Terraform自动化部署一个CSS集群，包括创建VPC、子网、安全组，以及配置集群引擎类型、安全模式、HTTPS访问和计费方式等参数。
* [部署Logstash集群](logstash_cluster.md) - 介绍如何使用Terraform自动化部署一个CSS Logstash集群，包括创建VPC、子网、安全组，以及配置节点规格、存储和计费方式等参数。

## 参考资料

- [华为云CSS产品文档](https://support.huaweicloud.com/css/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
