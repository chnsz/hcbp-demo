# Introduction

## What is NAT Gateway

NAT Gateway is a high-performance, high-availability network address translation service provided by Huawei Cloud. It enables resources in a Virtual Private Cloud (VPC), such as Elastic Cloud Servers and Bare Metal Servers, to access the internet and be accessed from the internet. Through SNAT and DNAT, NAT Gateway interconnects cloud resources with the public network without changing their private IP addresses, helping enterprises build secure and cost-effective cloud network entrances and exits.

NAT Gateway supports SNAT (Source Network Address Translation), allowing multiple cloud servers in a VPC to share an elastic IP to proactively access the internet without binding a public IP to each server. It also supports DNAT (Destination Network Address Translation), which maps access traffic on a public EIP to the private IP address and port of a backend server in the VPC by port, so that internal services can be securely exposed to the internet. NAT Gateway is available in multiple specifications that can be selected based on traffic scale and scaled on demand.

With NAT Gateway, enterprises can centrally manage the public network entrances and exits of cloud resources, reduce public IP consumption and security exposure, and combine security groups and network ACLs to build a layered network security protection system, laying a solid foundation for subsequent network O&M and cost optimization.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud NAT Gateway, helping you understand how to efficiently manage cloud NAT Gateway resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for NAT Gateway resources. These best practices will help you quickly get started with automated NAT Gateway deployment and lay a solid foundation for subsequent VPC, subnet, elastic IP, and ECS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy DNAT Rule](dnat_basic.md) - Introduces how to use Terraform to automatically deploy a DNAT rule, including VPC and subnet creation, NAT gateway creation, elastic IP creation, backend ECS instance and security group configuration, and DNAT port mapping configuration.

## Reference Materials

- [Huawei Cloud NAT Gateway Product Documentation](https://support.huaweicloud.com/nat/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
