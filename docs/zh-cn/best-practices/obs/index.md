# 简介

## 什么是对象存储服务（OBS）

对象存储服务（Object Storage Service，OBS）是华为云提供的高可用、高可靠、高性能、安全、低成本的对象存储服务。OBS提供海量、安全、高可靠、低成本的数据存储能力，支持多种存储类型，包括标准存储、低频存储、归档存储等，满足不同业务场景的存储需求。

通过OBS，您可以存储和管理任意类型的数据，包括文档、图片、视频、音频等。华为云OBS支持多种安全功能，包括服务端加密、访问控制、审计日志等，为企业提供安全可靠的数据存储解决方案。华为云OBS支持与ECS、FunctionGraph、CDN等服务的无缝集成，为您提供完整的云存储解决方案。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云对象存储服务（OBS）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的存储资源。

通过本章节的最佳实践，您可以学习到主要的对象存储资源的部署流程，这些最佳实践将帮助您快速上手对象存储的自动化部署，并为后续的OBS管理和运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署KMS加密桶](kms_encrypted_bucket.md) - 介绍如何使用Terraform自动化部署OBS KMS加密桶，包括KMS密钥创建和OBS桶配置。
* [部署桶静态网站托管配置](static_website_hosting.md) - 介绍如何使用Terraform自动化部署OBS桶静态网站托管配置，包括KMS密钥创建、OBS桶配置、网站配置和访问策略设置。
* [部署通过静态文本上传的桶对象](object_upload_with_content.md) - 介绍如何使用Terraform自动化部署通过静态文本上传的桶对象，包括KMS密钥创建、OBS桶配置和对象上传。
* [部署通过加密文本上传的桶对象](object_upload_with_encryption.md) - 介绍如何使用Terraform自动化部署通过加密文本上传的桶对象，包括KMS密钥创建、OBS桶配置、文件压缩和加密上传。

## 参考资料

- [华为云OBS产品文档](https://support.huaweicloud.com/obs/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
