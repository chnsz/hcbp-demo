# Introduction

## What is Bare Metal Server (BMS)

Bare Metal Server (BMS) is a dedicated physical server provided by Huawei Cloud, offering secure and reliable dedicated computing resources without virtualization overhead, meeting business scenarios with high performance requirements such as high-performance computing, databases, and big data analysis. BMS provides complete control over physical servers, supports custom operating systems, network configurations, and security policies, and is suitable for application scenarios with strict requirements for performance, security, and compliance.

BMS service supports multiple instance types, providing different computing capabilities, storage space, and network performance to meet different business needs. BMS supports VPC and security group isolation, providing secure and reliable dedicated computing resources to meet enterprise requirements for data security and compliance. Through BMS, enterprises can obtain the complete performance and isolation of physical servers while enjoying the flexibility and convenience of cloud computing.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Bare Metal Server (BMS), helping you understand how to efficiently manage cloud physical server resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for bare metal server resources. These best practices will help you quickly get started with automated bare metal server deployment and lay a solid foundation for subsequent BMS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Basic Instance](basic.md) - Introduces how to use Terraform to automatically deploy a basic BMS instance, including the creation of VPC, subnet, security group, and key pair.
* [Deploy Reset Instance Password](reset_password.md) - Introduces how to use Terraform to automatically reset the password of a BMS instance, ensuring standardized and secure password management.
* [Deploy Attach Volume](volume_attach.md) - Introduces how to use Terraform to automatically attach a cloud disk to a BMS instance, expanding the instance's storage capacity.

## Reference Materials

- [Huawei Cloud BMS Product Documentation](https://support.huaweicloud.com/bms/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
