# Introduction

## What is Enterprise Switch (ESW)

Enterprise Switch (ESW) is a high-performance, highly available enterprise-grade network switching service provided by Huawei Cloud, supporting large Layer 2 network interconnection and achieving Layer 2 network connections across availability zones. ESW service provides virtualized network switching capabilities, supporting Layer 2 network interconnection within VPCs and across VPCs, meeting enterprise-grade network requirements.

ESW service supports high-availability deployment modes, providing primary and standby availability zone deployment to ensure high availability of network services. Through ESW service, enterprises can achieve Layer 2 network interconnection across availability zones, supporting scenarios such as virtual machine migration, load balancing, and high-availability deployment, meeting enterprise-grade network architecture requirements.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Enterprise Switch (ESW), helping you understand how to efficiently manage cloud ESW resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for ESW resources. These best practices will help you quickly get started with automated ESW deployment and lay a solid foundation for subsequent ESW management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Connection with Virtual Port Binding](connection_vport_bind.md) - Introduces how to use Terraform to automatically deploy ESW connection with virtual port binding, including VPC, subnet, ESW instance, ESW connection, VPC subnet private IP, and connection virtual port binding creation.

## Reference Materials

- [Huawei Cloud ESW Product Documentation](https://support.huaweicloud.com/esw/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
