# Introduction

## What is Cloud Container Engine (CCE)

Cloud Container Engine (CCE) is a high-reliability, high-performance enterprise-grade container management service that supports Kubernetes community native applications and tools. It provides multiple types of container clusters, making it convenient for users to deploy, manage, and scale containerized applications. Through CCE, you can easily build Kubernetes-based containerized applications and implement microservices architecture.

CCE supports two cluster types: Standard and Turbo, providing full lifecycle management, supporting multiple types of nodes including ECS and BMS, and providing elastic scaling capabilities. It supports VPC networks and container tunnel networks, enabling efficient communication between containers, and provides multiple storage types to meet different application scenario requirements.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Cloud Container Engine (CCE), helping you understand how to efficiently manage cloud container resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for container cluster resources. These best practices will help you quickly get started with automated Cloud Container Engine deployment and lay a solid foundation for subsequent CCE management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Node Pool](node_pool.md) - Introduces how to use Terraform to automatically deploy a CCE node pool, including querying availability zones and instance flavors, as well as creating VPC, subnet, Elastic IP, CCE cluster, key pair, and node pool.
* [Deploy Standard Cluster](standard_cluster.md) - Introduces how to use Terraform to automatically deploy a CCE Standard cluster, including querying availability zones, as well as creating VPC, subnet, Elastic IP, and CCE cluster.
* [Deploy Turbo Cluster](turbo_cluster.md) - Introduces how to use Terraform to automatically deploy a CCE Turbo cluster, including querying availability zones, as well as creating VPC, subnet, ENI subnet, Elastic IP, and CCE cluster.

## Reference Materials

- [Huawei Cloud CCE Product Documentation](https://support.huaweicloud.com/cce/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
