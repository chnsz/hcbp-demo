# Introduction

## What is Resource Governance Center (RGC)

Resource Governance Center (RGC) is a resource governance service provided by Huawei Cloud, supporting multi-account management, organizational unit management, blueprint configuration, and other functions to help you uniformly manage and govern cloud resources. RGC service provides centralized resource management capabilities, supporting cross-account resource governance, compliance checks, and automated deployment, meeting enterprise-level resource management and compliance requirements.

RGC service supports hierarchical management of organizational units (OU). By creating organizational units, accounts can be grouped for management, achieving unified resource governance. Through blueprint configuration functionality, automated resource deployment and management can be achieved, improving the efficiency and convenience of resource management. Through RGC service, enterprises can achieve unified governance of cloud resources, ensure resource security and compliance, and improve the efficiency and standardization of resource management.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Resource Governance Center (RGC), helping you understand how to efficiently manage cloud RGC resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for RGC resources. These best practices will help you quickly get started with automated RGC deployment and lay a solid foundation for subsequent account management, organizational unit management, and blueprint configuration operations.

## Best Practices List

This section contains the following best practices:

* [Deploy Account](account.md) - Introduces how to use Terraform to automatically deploy RGC accounts, including account basic information, identity store information, and organizational unit information configuration.
* [Deploy Account Enroll](account_enroll.md) - Introduces how to use Terraform to automatically deploy RGC account enrollment, including creating organizational units (optional) and account enrollment (with blueprint configuration).
* [Deploy Template](template.md) - Introduces how to use Terraform to automatically deploy RGC templates, including the creation of predefined templates and customized templates.

## Reference Materials

- [Huawei Cloud RGC Product Documentation](https://support.huaweicloud.com/rgc/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
