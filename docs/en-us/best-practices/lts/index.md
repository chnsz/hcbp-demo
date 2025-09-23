# Introduction

## What is Log Tank Service (LTS)

Log Tank Service (LTS) is a one-stop log management service provided by Huawei Cloud, designed to help users efficiently collect, store, query, and analyze log data. LTS supports multiple log collection methods, provides powerful log search and analysis capabilities, and can meet various enterprise-level log management needs. LTS service features high reliability, high performance, and ease of use, supporting real-time log monitoring, alert notifications, log auditing, and other functions.

As one of Huawei Cloud's core log management services, LTS supports seamless integration with ECS, CCE, APIG, FunctionGraph, and other services, meeting complex log management scenario requirements. LTS provides flexible log group and stream management capabilities, supporting advanced functions such as log lifecycle management and tag classification.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Log Tank Service (LTS), helping you understand how to efficiently manage cloud log management resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for log management resources. These best practices will help you quickly get started with automated LTS deployment and lay a solid foundation for subsequent LTS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Log Stream](log_stream.md) - Introduces how to use Terraform to automatically deploy LTS log streams, including log group and log stream creation, supporting log lifecycle management and tag classification functions.
* [Deploy Log Transfer](log_transfer.md) - Introduces how to use Terraform to automatically deploy LTS log transfer, including log group, log stream, OBS bucket, and log transfer task creation, supporting long-term storage and backup of log data.
* [Deploy SQL Alarm Rule](sql_alarm_rule.md) - Introduces how to use Terraform to automatically deploy LTS SQL alarm rules, including SMN topic, log group, log stream, and SQL alarm rule creation, supporting real-time monitoring and alert notifications based on SQL query results.

## Reference Materials

- [Huawei Cloud Log Tank Service Product Documentation](https://support.huaweicloud.com/lts/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
