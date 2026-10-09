# Introduction

## What is Cloud Container Engine (CCE) Autopilot

Cloud Container Engine (CCE) Autopilot is a serverless Kubernetes cluster mode provided by Huawei Cloud, delivering O&M-free container orchestration for cloud-native applications. Autopilot clusters fully manage the control plane and node resources, so you do not need to create or manage nodes; you only need to submit workloads to run containerized applications, allowing you to focus on business development and innovation.

CCE Autopilot clusters use the ENI container network mode and support Kubernetes community-native APIs and tools, remaining compatible with the mainstream cloud-native ecosystem. Clusters include built-in capabilities such as auto scaling, load balancing, logging, and monitoring, and support installing add-ons for log collection, monitoring, and networking from the add-on marketplace, helping you quickly build an observable and scalable containerized application platform.

With CCE Autopilot, enterprises can pay based on actual computing resource usage, reducing cluster O&M costs and idle resources. At the same time, the add-on capabilities help improve logging, monitoring, and auditing systems, laying a solid foundation for subsequent containerized business management and operation work.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Cloud Container Engine (CCE) Autopilot, helping you understand how to efficiently manage cloud CCE Autopilot resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for CCE Autopilot resources. These best practices will help you quickly get started with automated CCE Autopilot deployment and lay a solid foundation for subsequent cluster, add-on, network management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy CCE Autopilot Addon](cce_autopilot_addons.md) - Introduces how to use Terraform to automatically deploy a CCE Autopilot cluster add-on, including VPC and subnet creation, CCE Autopilot cluster creation, SWR organization creation, and log-agent add-on deployment.
* [Deploy CCE Autopilot Application](cce_autopilot_chart_release.md) - Introduces how to use Terraform to automatically deploy a CCE Autopilot application, including VPC and subnet creation, CCE Autopilot cluster creation, Helm chart upload, and Helm release deployment.

## Reference Materials

- [Huawei Cloud Cloud Container Engine Product Documentation](https://support.huaweicloud.com/cce/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
