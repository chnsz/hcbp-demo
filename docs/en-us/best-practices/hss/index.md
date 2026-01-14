# Introduction

## What is Host Security Service (HSS)

Host Security Service (HSS) is a host security protection service provided by Huawei Cloud, offering asset management, vulnerability management, intrusion detection, baseline checks, and other functions to help you comprehensively protect the security of cloud hosts. HSS service supports multiple operating systems, including Linux and Windows, providing real-time monitoring, threat detection, security hardening, and other capabilities, meeting enterprise-grade host security protection requirements.

HSS service provides host group management functionality, supporting grouping of multiple hosts for management, uniformly configuring security policies, performing security checks, and conducting security operations. Through HSS service, enterprises can achieve unified management and monitoring of host security, improve security operation efficiency, and ensure the safe and stable operation of business systems.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Host Security Service (HSS), helping you understand how to efficiently manage cloud HSS resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for HSS resources. These best practices will help you quickly get started with automated HSS deployment and lay a solid foundation for subsequent host group management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Host Group](host_group.md) - Introduces how to use Terraform to automatically deploy HSS host groups, including VPC, subnet, security group, ECS instance (with HSS agent), and HSS host group creation.
* [Deploy Host Protection](host_protection.md) - Introduces how to use Terraform to automatically deploy HSS host protection, enabling pay-per-use HSS protection services for existing hosts.

## Reference Materials

- [Huawei Cloud HSS Product Documentation](https://support.huaweicloud.com/hss/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
