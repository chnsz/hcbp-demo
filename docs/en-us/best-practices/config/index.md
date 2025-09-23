# Introduction

## What is Config (Named as Resource Management Service (RMS) before)

Config, named as Resource Management Service (RMS) before, it is a one-stop compliance management service provided by Huawei Cloud, helping users continuously monitor and evaluate the configuration compliance of cloud resources. Config service provides pre-built compliance rule packages and custom rules, supporting multiple compliance frameworks and standards, helping enterprises establish a comprehensive compliance management system.

Through Config, you can monitor configuration changes of cloud resources, evaluate whether resource configurations meet security and compliance requirements, and promptly discover and fix configuration risks. Config service supports multiple compliance frameworks, including international standards such as CIS, NIST, SOC, as well as Huawei Cloud security best practices, providing enterprises with comprehensive compliance management solutions. Huawei Cloud Config supports seamless integration with cloud monitoring, message notification, and other services, providing you with a complete compliance monitoring and alerting system.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Configuration Audit (Config), helping you understand how to efficiently manage cloud compliance resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for configuration audit resources. These best practices will help you quickly get started with automated configuration audit deployment and lay a solid foundation for subsequent Config management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Compliance Package](compliance_package.md) - Introduces how to use Terraform to automatically deploy Config compliance packages, including rule package template queries and compliance package creation.
* [Deploy Compliance Rule](compliance_rule.md) - Introduces how to use Terraform to automatically deploy Config compliance rules, including VPC creation, ECS instance creation, compliance rule configuration, and rule evaluation.
* [Deploy Resource Aggregator](resource_aggregator.md) - Introduces how to use Terraform to automatically deploy Config resource aggregators, including aggregator creation and account configuration.

## Reference Materials

- [Huawei Cloud Config Product Documentation](https://support.huaweicloud.com/rms/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
