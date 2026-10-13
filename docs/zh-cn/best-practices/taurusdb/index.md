# 简介

## 什么是云数据库TaurusDB（TaurusDB）

云数据库TaurusDB是华为云提供的企业级云原生数据库服务，完全兼容MySQL协议，采用计算与存储分离的架构设计。TaurusDB将计算节点与存储节点解耦，支持存储按需扩展、计算节点秒级弹性伸缩，能够有效应对业务负载波动，帮助企业在云上构建高性能、高可靠的数据库服务。

TaurusDB提供一写多读能力，单个实例最多可挂载多个只读节点，读请求可在多个只读节点之间自动负载均衡，显著提升高并发读场景下的处理能力。同时，TaurusDB支持并行查询、秒级监控、SQL审计、参数模板、备份恢复等企业级能力，满足在线事务处理（OLTP）等对性能、可靠性和扩展性要求较高的业务场景。

借助TaurusDB，企业无需自行搭建和维护数据库集群，即可获得接近原生MySQL的使用体验与更高的性能表现，从而将更多精力投入到业务开发与创新之中。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云云数据库TaurusDB（TaurusDB）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的TaurusDB资源。

通过本章节的最佳实践，您可以学习到主要的TaurusDB资源的部署流程，这些最佳实践将帮助您快速上手TaurusDB的自动化部署，并为后续的TaurusDB实例、账号、数据库管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署TaurusDB实例](taurusdb_instance.md) - 介绍如何使用Terraform自动化部署TaurusDB实例，包括VPC创建、子网创建、安全组配置、参数模板、实例配置、账号和数据库管理。

## 参考资料

- [华为云TaurusDB产品文档](https://support.huaweicloud.com/taurusdb/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
