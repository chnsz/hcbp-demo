# 简介

## 什么是消息通知服务（SMN）

消息通知服务（Simple Message Notification，SMN）是华为云提供的可靠、可扩展的消息通知服务，支持多种消息通知方式，包括邮件、短信、HTTP/HTTPS等。SMN服务提供统一的消息发布和订阅机制，支持主题订阅、消息推送、消息过滤等功能，帮助您实现应用与用户之间的消息通信。

SMN服务支持多种订阅协议，包括邮件、短信、HTTP/HTTPS、函数工作流等，满足不同场景的消息通知需求。通过SMN服务，您可以实现消息的可靠传递、批量推送和灵活订阅，构建高效的消息通知系统。SMN服务还支持与云监控服务（CES）集成，实现告警通知的自动化发送。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云消息通知服务（SMN）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的SMN资源。

通过本章节的最佳实践，您可以学习到主要的SMN资源的部署流程，这些最佳实践将帮助您快速上手SMN的自动化部署，并为后续的主题管理、订阅管理和告警通知运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署CES事件告警规则](ces_event_alarm_rule.md) - 介绍如何使用Terraform自动化部署CES事件告警规则，包括创建SMN主题和配置CES告警规则，实现SMN主题事件的监控和告警。
* [部署消息发布](publish_message.md) - 介绍如何使用Terraform自动化部署消息发布，包括创建SMN主题、订阅、消息模板（可选）和发布消息。

## 参考资料

- [华为云SMN产品文档](https://support.huaweicloud.com/smn/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
