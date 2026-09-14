# Introduction

## What is Elastic IP (EIP)

Elastic IP (EIP) is an independently applicable and bindable public IP address resource provided by Huawei Cloud, offering cloud resources the ability to access the public network and be accessed from it. An EIP can be flexibly bound to and unbound from cloud resources such as Elastic Cloud Servers, Bare Metal Servers, Elastic Load Balancers, NAT Gateways, and virtual IP addresses, helping users build public access entries on demand without configuring a fixed public address for each server.

EIP supports bandwidth-based and traffic-based billing modes, and can be used together with shared bandwidth. Shared bandwidth allows multiple EIPs to be added to the same bandwidth resource, enabling bandwidth reuse and sharing, thereby reducing public bandwidth costs and improving bandwidth utilization. EIP also supports binding enterprise projects, setting tags, and configuring public border groups, meeting enterprise-level resource management and cost allocation requirements.

With EIP, users can flexibly provide stable, secure, and manageable public access capabilities for cloud workloads, and combine features such as shared bandwidth and bandwidth adjustment to achieve elastic scaling and cost optimization of public network egress.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Elastic IP (EIP), helping you understand how to efficiently manage cloud EIP resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for EIP resources. These best practices will help you quickly get started with automated EIP deployment and lay a solid foundation for subsequent shared bandwidth, public access management, and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy EIP Bound to Shared Bandwidth](eip_associate_shared_bandwidth.md) - Introduces how to use Terraform to automatically deploy an EIP bound to shared bandwidth, including shared bandwidth creation, elastic IP creation, and EIP-to-shared-bandwidth association management.

## Reference Materials

- [Huawei Cloud Elastic IP Product Documentation](https://support.huaweicloud.com/eip/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
