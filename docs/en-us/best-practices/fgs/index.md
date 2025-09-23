# Introduction

## What is FunctionGraph

FunctionGraph is an event-driven serverless computing service. It provides three development approaches: zero-code, low-code, and custom code, supporting multiple programming languages. You can run tasks elastically and reliably without managing infrastructure such as servers, just by writing business function code and setting execution conditions. Through FunctionGraph, you can quickly build flexible data processing workflows and application integration solutions.

As one of Huawei Cloud's core computing services, FunctionGraph supports multiple trigger types, including timer triggers, OBS triggers, SMN triggers, APIG triggers, DMS triggers, etc., meeting the needs of different business scenarios. FunctionGraph provides cloud computing advantages such as elastic scaling, pay-as-you-go, and high reliability, allowing you to focus on business logic development without worrying about underlying infrastructure management.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud FunctionGraph, helping you understand how to efficiently manage cloud serverless computing resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for serverless computing resources. These best practices will help you quickly get started with automated FunctionGraph deployment and lay a solid foundation for subsequent FunctionGraph management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy CTS Trigger](cts_trigger.md) - Implement cloud resource operation monitoring and response through CTS triggers, supporting security auditing, compliance monitoring, automated operations, and other functions, providing complete Terraform configuration and parameter descriptions.
* [Deploy EG Trigger](eg_trigger.md) - Implement event-driven image processing through EG triggers, supporting automatic thumbnail generation when OBS files are uploaded, providing complete Terraform configuration and parameter descriptions.
* [Deploy Timer Trigger](timer_trigger.md) - Implement periodic task execution through timer triggers, supporting both Cron expression and fixed interval scheduling methods, providing complete Terraform configuration and parameter descriptions.

## Reference Materials

- [Huawei Cloud FunctionGraph Product Documentation](https://support.huaweicloud.com/functiongraph/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
