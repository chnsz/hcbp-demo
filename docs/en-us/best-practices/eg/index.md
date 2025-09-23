# Introduction

## What is EventGrid (EG)

EventGrid (EG) is an event-driven architecture service provided by Huawei Cloud, supporting event production, routing, transformation, and consumption. Through EventGrid, you can build loosely coupled, scalable distributed application systems, implementing event-driven microservice architectures.

EventGrid supports multiple event sources and targets, including cloud service events, custom events, third-party system events, etc., providing flexible event filtering, transformation, and routing capabilities. It supports event subscription, event channel management, event source configuration, and other functions, helping enterprises implement efficient event-driven architectures.

As one of Huawei Cloud's core integration services, EventGrid supports seamless integration with FunctionGraph, OBS, SMN, APIG, and other services, meeting complex event-driven scenario requirements. EventGrid provides high-reliability, low-latency event delivery capabilities, supporting advanced features such as event retry and dead letter queues.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud EventGrid (EG), helping you understand how to efficiently manage cloud event-driven resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for event-driven resources. These best practices will help you quickly get started with automated EventGrid deployment and lay a solid foundation for subsequent EG management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Event Subscription (Custom Event Source, EG Event Target)](event_subscription_custom_to_eg.md) - Introduces how to use Terraform to automatically deploy event subscriptions, implementing event subscription and routing through custom event sources and EG event targets, supporting event filtering, transformation, and distribution functions.
* [Deploy Event Subscription (OBS Event Source, Kafka Event Target)](event_subscription_obs_to_kafka.md) - Introduces how to use Terraform to automatically deploy event subscriptions, implementing event subscription and routing through OBS event sources and Kafka event targets, supporting real-time push of OBS storage operation events and Kafka message queue integration.
* [Deploy Event Subscription (VPC Event Source, EG Event Target)](event_subscription_vpc_to_eg.md) - Introduces how to use Terraform to automatically deploy event subscriptions, implementing event subscription and routing through VPC event sources and EG event targets, supporting real-time push of VPC network operation events and event forwarding between EG channels.

## Reference Materials

- [Huawei Cloud EventGrid Product Documentation](https://support.huaweicloud.com/eg/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
