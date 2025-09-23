# Introduction

## What is Enterprise Router (ER)

Enterprise Router (ER) is a high-performance, highly available enterprise-grade router service provided by Huawei Cloud, supporting enterprise-level network functions such as multi-VPC interconnection, dedicated line access, and VPN connections. As one of the core components of Huawei Cloud's network services, ER provides flexible routing policies and rich network connectivity capabilities, meeting complex enterprise network architecture requirements.

Through ER, you can build enterprise-grade network architectures, implementing functions such as multi-VPC interconnection, dedicated line access, and VPN connections, enjoying enterprise-level network advantages such as high performance, high reliability, and flexible scaling. Huawei Cloud ER supports seamless integration with VPC, dedicated lines, VPN, and other services, providing you with complete network solutions.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Enterprise Router (ER), helping you understand how to efficiently manage cloud network resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for enterprise router resources. These best practices will help you quickly get started with automated ER deployment and lay a solid foundation for subsequent ER management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Flow Logs](flow_log.md) - Introduces how to use Terraform to automatically deploy flow logs, including VPC creation, ER instance creation, VPC connection, LTS log group creation, and flow log configuration.
* [Deploy Route Table](route_table.md) - Introduces how to use Terraform to automatically deploy route tables, including VPC creation, ER instance creation, and route table configuration.
* [Deploy VPC Connection](vpc_attachment.md) - Introduces how to use Terraform to automatically deploy VPC connections, including VPC creation, ER instance creation, and VPC connection configuration.
* [Deploy Shared Instance](share_instance.md) - Introduces how to use Terraform to automatically deploy ER shared instances, including ER instance creation, RAM resource sharing, cross-account VPC connections, and attachment acceptance.

## Reference Materials

- [Huawei Cloud ER Product Documentation](https://support.huaweicloud.com/er/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
