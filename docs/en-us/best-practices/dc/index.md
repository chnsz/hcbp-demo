# Introduction

## What is Direct Connect (DC)

Direct Connect (DC) is a high-performance, low-latency, secure, and reliable dedicated line access service provided by Huawei Cloud, offering enterprises dedicated network connections from local data centers to Huawei Cloud. Direct Connect service supports multiple access methods, including physical dedicated lines and virtual dedicated lines, meeting network connection requirements for different scales and scenarios.

Direct Connect service provides complete dedicated line lifecycle management functionality, supporting enterprise-level features such as dedicated line monitoring, fault diagnosis, and performance optimization, with high reliability and security. Through Direct Connect service, enterprises can easily implement hybrid cloud architecture, ensure data transmission security and stability, and reduce network latency and costs.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Direct Connect (DC), helping you understand how to efficiently manage cloud dedicated line resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DC resources. These best practices will help you quickly get started with automated Direct Connect deployment and lay a solid foundation for subsequent dedicated line management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Connect Gateway](connect_gateway.md) - Introduces how to use Terraform to automatically deploy DC connect gateways, including gateway creation and configuration.
* [Deploy Global Gateway](global_gateway.md) - Introduces how to use Terraform to automatically deploy DC global gateways, including gateway creation, BGP configuration, and tag management.
* [Deploy Hosted Connect](hosted_connect.md) - Introduces how to use Terraform to automatically deploy DC hosted connects, including connection creation, bandwidth configuration, and VLAN allocation.
* [Deploy Virtual Interface](virtual_interface.md) - Introduces how to use Terraform to automatically deploy DC virtual interfaces, including VPC creation, virtual gateway creation, and virtual interface configuration.

## Reference Materials

- [Huawei Cloud DC Product Documentation](https://support.huaweicloud.com/dc/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
