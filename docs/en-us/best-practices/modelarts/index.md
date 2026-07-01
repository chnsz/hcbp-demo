# Introduction

## What is ModelArts

ModelArts is a model training and inference platform provided by Huawei Cloud for AI developers, offering end-to-end AI development capabilities from data processing, algorithm development, model training to model deployment. ModelArts supports multiple deep learning frameworks and provides Notebook development environments, distributed training, AutoLearning, model management, and other features to help developers quickly build, train, and deploy AI models.

ModelArts provides multiple compute resource management modes such as dedicated resource pools and on-demand resource pools, and supports deep integration with cloud services such as VPC and SFS Turbo to meet enterprise AI training and inference requirements for network isolation, high-performance storage, and elastic computing power. With ModelArts, users can automate the end-to-end AI workflow from data preparation to model deployment.

## Best Practices Overview

This chapter provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud ModelArts, helping you understand how to efficiently manage cloud AI training and inference resources using Infrastructure as Code (IaC).

Through the best practices in this chapter, you can learn the deployment workflows of major ModelArts resources. These best practices will help you quickly get started with automated ModelArts deployment and lay a solid foundation for subsequent training job, dedicated resource pool, and network management operations.

## Best Practice List

This chapter includes the following best practices:

* [Deploy Custom Training Job with Dedicated Resource Pool](custom_training_job_with_dedicated_resource_pool.md) - Introduces how to use Terraform to automatically deploy a ModelArts dedicated resource pool and custom training job, including VPC network, SFS Turbo, ModelArts network, workspace, dedicated resource pool, and training job creation.
* [Deploy Dedicated Resource Pool](dedicated_resource_pool.md) - Introduces how to use Terraform to automatically deploy a ModelArts dedicated resource pool, supporting creation of new or reuse of existing ModelArts networks and SFS Turbo, including VPC network, SFS Turbo, ModelArts network, workspace, and dedicated resource pool creation.
* [Deploy Real-Time Inference Service](deploy_real_time_service.md) - Introduces how to use Terraform to automatically deploy a ModelArts V2 real-time inference service, including V2 resource pool query, instance group and unit configuration, runtime configuration, upgrade strategy, and log configuration.
* [Deploy Fine-Tuning Training Job](fine_tuning_training_job.md) - Introduces how to use Terraform to automatically deploy a ModelArts fine-tuning training job, including workspace, SMN topic, and fine-tuning training job creation, supporting dataset inputs, asset model, output model, and fine-tuning configuration.
* [Deploy Notebook with Dedicated Resource Pool](notebook_with_dedicated_resource_pool.md) - Introduces how to use Terraform to automatically deploy a ModelArts dedicated resource pool and Notebook instance, including VPC network, SFS Turbo, ModelArts network, workspace, dedicated resource pool, Notebook flavor and image queries, and storage mount creation.

## References

- [Huawei Cloud ModelArts Product Documentation](https://support.huaweicloud.com/modelarts/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
