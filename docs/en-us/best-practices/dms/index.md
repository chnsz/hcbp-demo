# Introduction

## What is Distributed Message Service (DMS)

Huawei Cloud Distributed Message Service (DMS) is a highly available, highly reliable, and high-performance distributed message middleware service that is fully compatible with open-source RocketMQ, RabbitMQ, and Kafka, providing message sending and receiving, message storage, message routing, and other functions.

DMS supports multiple message patterns, including point-to-point and publish-subscribe, meeting message communication requirements for different business scenarios. And provides three message engines: RocketMQ, RabbitMQ, and Kafka, supporting both cluster and single-node deployment modes with high availability, high performance, and high reliability characteristics. Through DMS, enterprises can build loosely coupled, scalable distributed system architectures, implementing modern application patterns such as asynchronous message processing and event-driven architecture.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Distributed Message Service (DMS), helping you understand how to efficiently manage cloud message middleware resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for message middleware resources. These best practices will help you quickly get started with automated Distributed Message Service deployment and lay a solid foundation for subsequent DMS management and operation work.

## Best Practices List

This section contains the following best practices:

### RocketMQ Best Practices

* [Deploy RocketMQ Basic Instance](rocketmq/basic_instance.md) - Introduces how to use Terraform to automatically deploy a basic RocketMQ instance, including VPC, subnet, security group, and EIP creation.
* [Deploy RocketMQ Consumer Group](rocketmq/consumer_group.md) - Introduces how to use Terraform to automatically deploy a RocketMQ consumer group, including RocketMQ instance and consumer group creation.
* [Deploy RocketMQ Message Send](rocketmq/message_send.md) - Introduces how to use Terraform to automatically deploy RocketMQ message sending functionality, including RocketMQ instance, topic, and message sending creation.
* [Deploy RocketMQ Migration Task](rocketmq/migration_task.md) - Introduces how to use Terraform to automatically deploy RocketMQ migration tasks, including RocketMQ instance and migration task creation.

## Reference Materials

* [Huawei Cloud Distributed Message Service RocketMQ Product Documentation](https://support.huaweicloud.com/hrm/index.html)
* [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
