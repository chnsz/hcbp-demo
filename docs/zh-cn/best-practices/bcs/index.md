# 简介

## 什么是区块链服务（BCS）

区块链服务（Blockchain Service，简称BCS）是面向企业及开发者提供的区块链技术服务平台，可以帮助您快速部署、管理、维护区块链网络，降低使用区块链的门槛，让您专注于自身业务的开发与创新，实现业务快速上链。

BCS支持创建Hyperledger Fabric增强版和华为云区块链引擎实例，提供用户管理、节点管理、运维监控等模块，帮助您快速创建、方便管理、高效运维区块链网络，为上层应用提供企业级区块链系统，可应用于供应链金融、供应链溯源、数字资产等多种业务场景。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云区块链服务（BCS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的BCS资源。

通过本章节的最佳实践，您可以学习到主要的BCS资源的部署流程，这些最佳实践将帮助您快速上手BCS的自动化部署，并为后续的区块链网络管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署基础实例](basic.md) - 介绍如何使用Terraform自动化部署一个基础的BCS实例，包括指定服务版本、Fabric版本、共识算法、关联CCE集群，以及配置区块生成参数、SFS Turbo、Peer组织和通道。

## 参考资料

- [华为云BCS产品文档](https://support.huaweicloud.com/bcs/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
