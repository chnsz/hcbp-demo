# Introduction

## What is Elastic Load Balance (ELB)

Elastic Load Balance (ELB) is a service that automatically distributes access traffic to multiple cloud servers, enabling expansion of application system's external service capabilities and improving application availability. ELB automatically isolates abnormal backend servers through health checks to ensure high availability of services. Huawei Cloud ELB supports multiple load balancing algorithms, including round-robin, weighted round-robin, least connections, etc., to meet the needs of different business scenarios.

Elastic Load Balance provides two types: dedicated and shared, supporting multiple protocols such as TCP, UDP, HTTP, HTTPS, and can be seamlessly integrated with services such as ECS, VPC, and security groups. Through ELB, you can achieve traffic distribution, session persistence, health checks, and other functions, providing high-availability and high-performance load balancing services for application systems.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Elastic Load Balance (ELB), helping you understand how to efficiently manage cloud load balancing resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for Elastic Load Balance resources. These best practices will help you quickly get started with automated ELB deployment and lay a solid foundation for subsequent load balancer, listener, backend server group, and backend server management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Dedicated Load Balancer with Auto Scaling](dedicated_loadbalancer_with_as.md) - Introduces how to use Terraform to automatically deploy an integrated solution of dedicated load balancer and Auto Scaling, including VPC, subnet, dedicated load balancer, listener, backend server group, Auto Scaling configuration, Auto Scaling group, alarm rules, and scaling policies.
* [Deploy Dedicated Load Balancer with Backend Member](dedicated_loadbalancer_with_member.md) - Introduces how to use Terraform to automatically deploy a dedicated load balancer with backend members, including VPC, subnet, load balancer, listener, backend server group, health check, security group, ECS instance, and backend member creation.
* [Deploy Shared Load Balancer with Backend Member](shared_loadbalancer_with_member.md) - Introduces how to use Terraform to automatically deploy a shared load balancer with backend members, including VPC, subnet, load balancer, EIP binding, certificate management, listener, backend server group, health check, security group, security group rules, ECS instance, and backend member creation.

## Reference Materials

- [Huawei Cloud Elastic Load Balance Product Documentation](https://support.huaweicloud.com/elb/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
