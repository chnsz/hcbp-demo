# 简介

## 什么是分布式消息服务（DMS）

华为云分布式消息服务（Distributed Message Service，DMS）是一种高可用、高可靠、高性能的分布式消息中间件服务，完全兼容开源RocketMQ、RabbitMQ和Kafka，
提供消息收发、消息存储、消息路由等功能。DMS支持多种消息模式，包括点对点、发布订阅等，能够满足不同业务场景下的消息通信需求。

DMS提供RocketMQ、RabbitMQ和Kafka三种消息引擎，支持集群模式和单机模式部署，具备高可用、高性能、高可靠的特点。通过DMS，企业可以构建松耦合、可扩展的分布式系统架构，实现异步消息处理、事件驱动架构等现代化应用模式。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云分布式消息服务（DMS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的消息中间件资源。

通过本章节的最佳实践，您可以学习到主要的消息中间件资源的部署流程，这些最佳实践将帮助您快速上手分布式消息服务的自动化部署，并为后续的DMS管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

### RocketMQ最佳实践

* [部署RocketMQ基础实例](rocketmq/basic_instance.md) - 介绍如何使用Terraform自动化部署一个基础的RocketMQ实例，包括VPC、子网、安全组和EIP的创建。
* [部署RocketMQ消费者组](rocketmq/consumer_group.md) - 介绍如何使用Terraform自动化部署一个RocketMQ消费者组，包括RocketMQ实例和消费者组的创建。
* [部署RocketMQ消息发送](rocketmq/message_send.md) - 介绍如何使用Terraform自动化部署RocketMQ消息发送功能，包括RocketMQ实例、主题和消息发送的创建。
* [部署RocketMQ迁移任务](rocketmq/migration_task.md) - 介绍如何使用Terraform自动化部署RocketMQ迁移任务，包括RocketMQ实例和迁移任务的创建。

## 参考资料

* [华为云分布式消息服务RocketMQ产品文档](https://support.huaweicloud.com/hrm/index.html)
* [Terraform官方文档](https://www.terraform.io/docs/index.html)
