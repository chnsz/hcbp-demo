# Introduction

## What is NAT Gateway

NAT Gateway is a network address translation service provided by Huawei Cloud. It enables cloud resources in a Virtual Private Cloud (VPC) to access the internet or to be accessed from the internet through a public NAT gateway, without requiring each server to have a public IP address. NAT Gateway supports SNAT, which allows instances in a VPC to share an Elastic IP to access the internet, and DNAT, which maps a public EIP port to a port on a backend server so that external clients can reach internal services.

NAT Gateway provides multiple specifications ranging from small to extra-large, and supports flexible bandwidth configuration through Elastic IPs. It helps enterprises build secure, scalable, and cost-effective outbound and inbound access for cloud workloads, while keeping internal network addresses hidden from the public network.

By combining NAT Gateway with VPC, subnets, security groups, and ECS instances, you can implement complete public access architectures for scenarios such as web service exposure, remote login, and hybrid connectivity, and manage them consistently with Infrastructure as Code.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud NAT Gateway, helping you understand how to efficiently manage cloud NAT Gateway resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for NAT Gateway resources. These best practices will help you quickly get started with automated NAT Gateway deployment and lay a solid foundation for subsequent NAT Gateway management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy DNAT Rule](dnat_basic.md) - Introduces how to use Terraform to automatically deploy a DNAT rule, including VPC and subnet creation, NAT gateway creation, EIP creation, backend ECS instance and security group configuration, and DNAT port mapping.
* [Deploy NAT Gateway Shared via VPC Peering](nat_gateway_vpc_peering.md) - Introduces how to use Terraform to automatically deploy a NAT gateway shared across VPCs through VPC peering, including VPC and subnet creation, security group configuration, ECS instances, VPC peering connection and routes, EIPs, NAT gateway, SNAT rules, and DNAT rules.

## Reference Materials

- [Huawei Cloud NAT Gateway Product Documentation](https://support.huaweicloud.com/natgateway/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
