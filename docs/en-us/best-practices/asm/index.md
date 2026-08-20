# Introduction

## What is Application Service Mesh (ASM)

Application Service Mesh (ASM) provides a non-intrusive microservice governance solution with complete lifecycle management and traffic governance. It is compatible with the Kubernetes and Istio ecosystems, and supports capabilities such as load balancing, circuit breaking, and fault injection. It also includes built-in canary and blue-green release processes for one-stop automated release management.

Huawei Cloud ASM is deeply integrated with Cloud Container Engine (CCE) and provides service traffic management, service runtime monitoring, service access security, and service release capabilities as infrastructure. Both the control plane and data plane are compatible with open-source Istio, offering an out-of-the-box experience for customers.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Application Service Mesh (ASM), helping you understand how to efficiently manage ASM resources on the cloud using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for ASM resources. These best practices will help you quickly get started with automated ASM deployment and lay a solid foundation for subsequent mesh management and operations work.

## Best Practices List

This section contains the following best practices:

* [Deploy Basic Mesh](basic.md) - Introduces how to use Terraform to automatically deploy a basic ASM mesh associated with a single cluster, including specifying the mesh name, type, and version, and installing mesh components on the specified CCE cluster node.

## Reference Materials

- [Huawei Cloud ASM Product Documentation](https://support.huaweicloud.com/asm/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
