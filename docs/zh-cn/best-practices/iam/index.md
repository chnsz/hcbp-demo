# 简介

## 什么是统一身份认证服务（IAM）

统一身份认证服务（Identity and Access Management, IAM）是华为云提供的基础身份认证与访问管理服务，为华为云用户提供身份管理、权限管理和访问控制等核心功能。IAM服务支持细粒度的权限控制，通过用户、用户组、角色和策略的灵活组合，帮助企业实现统一、安全、高效的云资源访问管理。

IAM服务提供完整的身份认证和授权体系，支持多种认证方式、单点登录（SSO）、多因素认证（MFA）等安全功能，满足企业级身份管理和合规要求。通过IAM服务，企业可以实现最小权限原则，精细化控制用户对云资源的访问权限，提升云上资源的安全性和管理效率。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云统一身份认证服务（IAM）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的IAM资源。

通过本章节的最佳实践，您可以学习到主要的IAM资源的部署流程，这些最佳实践将帮助您快速上手IAM的自动化部署，并为后续的用户、用户组、角色和权限管理运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署用户组与策略关联](group_policies_associate.md) - 介绍如何使用Terraform自动化部署IAM用户组与策略关联，包括查询IAM策略、创建用户组和将策略关联到用户组。
* [部署密码策略](password_policy.md) - 介绍如何使用Terraform自动化部署IAM密码策略，包括密码长度、字符组合、有效期、重用规则等安全策略的配置。
* [通过用户组授权用户](users_authorized_through_group.md) - 介绍如何使用Terraform自动化部署IAM角色、用户组和用户，并通过用户组为用户授权，实现基于用户组的权限管理。

## 参考资料

- [华为云IAM产品文档](https://support.huaweicloud.com/iam/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
