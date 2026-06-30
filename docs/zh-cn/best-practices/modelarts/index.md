# 简介

## 什么是魔坊（ModelArts）

魔坊（ModelArts）是华为云提供的面向AI开发者的模型训推平台，提供从数据处理、算法开发、模型训练到模型部署的全流程AI开发能力。ModelArts支持多种深度学习框架，提供Notebook开发环境、分布式训练、自动学习、模型管理等功能，帮助开发者快速构建、训练和部署AI模型。

ModelArts提供专属资源池、按需资源池等多种计算资源管理方式，支持与VPC、SFS Turbo等云服务的深度集成，满足企业级AI训练与推理场景对网络隔离、高性能存储和弹性算力的需求。通过ModelArts，用户可以实现从数据准备到模型上线的端到端AI工作流自动化。

## 最佳实践简述

本章节提供了使用Terraform自动化部署和管理华为云魔坊（ModelArts）的最佳实践示例，帮助您了解如何利用Infrastructure as Code（IaC）的方式高效地管理云上的AI训练与推理资源。

通过本章节的最佳实践，您可以学习到主要的ModelArts资源的部署流程，这些最佳实践将帮助您快速上手ModelArts的自动化部署，并为后续的训练作业、专属资源池和网络管理等运维工作奠定坚实基础。

## 最佳实践列表

本章节包含以下最佳实践：

* [部署专属资源池自定义训练作业](custom_training_job_with_dedicated_resource_pool.md) - 介绍如何使用Terraform自动化部署ModelArts专属资源池及自定义训练作业，包括VPC网络、SFS Turbo、ModelArts网络、工作空间、专属资源池和训练作业的创建。
* [部署微调训练作业](fine_tuning_training_job.md) - 介绍如何使用Terraform自动化部署ModelArts微调训练作业，包括工作空间、SMN主题及微调训练作业的创建，支持数据集输入、资产模型、输出模型和微调配置。

## 参考资料

- [华为云ModelArts产品文档](https://support.huaweicloud.com/modelarts/index.html)
- [Terraform官方文档](https://www.terraform.io/docs/index.html)
