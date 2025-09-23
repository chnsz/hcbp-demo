# Introduction

## What is Cloud Trace Service (CTS)

Cloud Trace Service (CTS) is a security service provided by Huawei Cloud for recording all operations performed by users in cloud service accounts, including operations through the console, API, SDK, and other methods. CTS provides operation record query, audit, and problem location functions, supports saving operation records to OBS buckets, and meets security audit, compliance check, and other requirements.

As one of Huawei Cloud's core security services, CTS supports operation records for multiple cloud services, including ECS, VPC, OBS, RDS, etc., helping you achieve complete cloud operation auditing. CTS provides real-time monitoring, historical query, alert notification, and other functions, allowing you to comprehensively understand cloud resource usage and operation history.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Cloud Trace Service (CTS), helping you understand how to efficiently manage cloud audit and monitoring resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for audit and monitoring resources. These best practices will help you quickly get started with automated Cloud Trace Service deployment and lay a solid foundation for subsequent CTS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Data Tracker](data_tracker.md) - Implements automatic recording and storage of cloud resource operations through data trackers, supporting security audit, compliance monitoring, and other functions, providing complete Terraform configuration and parameter descriptions.
* [Deploy Notification](notification.md) - Implements real-time monitoring and alerting of cloud resource operations through CTS notifications, supporting security event monitoring, abnormal operation alerting, and other functions, providing complete Terraform configuration and parameter descriptions.
* [Deploy System Tracker](system_tracker.md) - Implements automatic recording and storage of cloud resource operations through system trackers, supporting security audit, compliance monitoring, and other functions, providing complete Terraform configuration and parameter descriptions.

## Reference Materials

- [Huawei Cloud CTS Product Documentation](https://support.huaweicloud.com/cts/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
