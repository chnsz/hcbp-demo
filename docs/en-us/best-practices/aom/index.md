# Introduction

## What is Application Operations Management (AOM)

Application Operations Management (AOM) is a one-stop application operations management platform provided by Huawei Cloud, offering enterprises a unified application operations management entry point. AOM helps enterprises achieve automated operations and intelligent management through application monitoring, log management, alarm management, and other functions, improving operational efficiency and quality.

AOM service supports multiple monitoring metrics and alarm rules, provides complete application lifecycle management capabilities, and supports seamless integration with services such as Log Tank Service and Simple Message Notification, helping enterprises build comprehensive operations management systems.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud AOM, helping you understand how to efficiently manage cloud application operations management resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for AOM resources. These best practices will help you quickly get started with automated AOM deployment and lay a solid foundation for subsequent AOM management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy AOM Alarm Action Callback](action_callback.md) - Introduces how to use Terraform to automatically deploy AOM alarm action callbacks, including creating SMN topics and subscriptions, AOM message templates, and configuring AOM alarm action rules.
* [Deploy AOM Distribute Alarms by Tags](distribute_alarm.md) - Introduces how to use Terraform to automatically deploy AOM distribute alarms by tags, including creating Prometheus instances, cloud service access, alarm rules, and configuring alarm action rules and tag management.
* [Deploy AOM Prevent ELB Alarm Storm](prevent_elb_alarm_storm.md) - Introduces how to use Terraform to automatically deploy AOM prevent ELB alarm storm, including creating LTS log groups and streams, SMN topics and log tanks, AOM alarm action rules, alarm group rules, and configuring alarm rules.

## Reference Materials

- [Huawei Cloud AOM Product Documentation](https://support.huaweicloud.com/aom/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
