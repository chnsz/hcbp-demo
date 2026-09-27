# Introduction

## What is NAT Gateway

NAT Gateway is a high-performance, high-availability public address translation service provided by Huawei Cloud, offering enterprises secure and convenient public network access capabilities. NAT Gateway supports both SNAT (Source Network Address Translation) and DNAT (Destination Network Address Translation) modes, helping cloud servers in a VPC access the Internet or be accessed from the Internet without binding an elastic IP, effectively reducing public IP resource costs and improving network security.

NAT Gateway provides multiple specifications, including small, medium, large, and extra-large, meeting business requirements of different scales and scenarios. Through SNAT rules, multiple cloud servers in a VPC can share one or more elastic IPs to access the Internet; through DNAT rules, a specified port of an elastic IP can be mapped to a specified port of a cloud server in the VPC to provide services externally. NAT Gateway also supports flexible combination with resources such as VPC, subnet, and elastic IP, helping users build secure and efficient cloud network architectures.

With infrastructure as code (IaC) tools such as Terraform, users can automatically create and manage NAT gateways and their rules, achieving versioned management and rapid deployment of network resources, and laying a solid foundation for subsequent network operation and expansion.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud NAT Gateway, helping you understand how to efficiently manage cloud NAT Gateway resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for NAT Gateway resources. These best practices will help you quickly get started with automated NAT Gateway deployment and lay a solid foundation for subsequent VPC, subnet, elastic IP, and cloud server management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy DNAT Rule](dnat_basic.md) - Introduces how to use Terraform to automatically deploy a DNAT rule, including VPC and subnet creation, NAT gateway creation, elastic IP creation, backend ECS instance creation, and DNAT rule configuration.

## Reference Materials

- [Huawei Cloud NAT Gateway Product Documentation](https://support.huaweicloud.com/natgateway/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
