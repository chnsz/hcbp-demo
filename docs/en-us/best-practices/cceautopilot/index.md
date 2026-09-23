# Introduction

## What is Cloud Container Engine (CCE) Autopilot

Cloud Container Engine (CCE) Autopilot is a serverless Kubernetes cluster form provided by Huawei Cloud, delivering a maintenance-free container runtime environment for cloud-native applications. Autopilot clusters shift the management and O&M of cluster nodes to Huawei Cloud, so you do not need to create, configure, or scale nodes and can focus on business workloads while obtaining highly reliable and elastic container runtime capabilities.

Autopilot clusters use the ENI container network mode and support on-demand elastic scaling and pay-per-use billing, automatically adjusting underlying resources based on business load to help you control costs while ensuring service stability. The cluster is compatible with native Kubernetes APIs and ecosystem tools, supports plug-in extension, and can meet observability and O&M requirements by installing add-ons such as log collection and monitoring.

With CCE Autopilot, enterprises can quickly build container platforms for microservices, online services, and batch processing scenarios, reducing cluster O&M complexity and improving resource utilization and service delivery efficiency.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Cloud Container Engine (CCE) Autopilot, helping you understand how to efficiently manage cloud CCE Autopilot resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for CCE Autopilot resources. These best practices will help you quickly get started with automated CCE Autopilot deployment and lay a solid foundation for subsequent cluster and add-on management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy CCE Autopilot Addon](cce_autopilot_addons.md) - Introduces how to use Terraform to automatically deploy a CCE Autopilot addon, including VPC creation, subnet creation, cluster creation, SWR organization creation, and log-agent add-on installation.

## Reference Materials

- [Huawei Cloud Cloud Container Engine Product Documentation](https://support.huaweicloud.com/cce/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
