# Introduction

## What is Global Accelerator (GA)

Global Accelerator (GA) is a global network acceleration service provided by Huawei Cloud. Leveraging Huawei Cloud's globally distributed points of presence and backbone network, GA provides nearby access and global acceleration capabilities for internet applications. By bringing user traffic into the Huawei Cloud backbone network from the nearest point of presence and forwarding it to origin servers over high-quality backbone links, GA effectively reduces latency and jitter for cross-region and cross-carrier access, improving the access experience and stability of services.

GA allows you to create accelerators with IP address sets, define the protocols and port ranges to be accelerated through listeners, and distribute traffic to origin servers in different regions through endpoint groups, enabling global traffic scheduling and nearby access. GA supports both IPv4 and IPv6 dual-stack access and can work with cloud resources such as Elastic Load Balance and Elastic Cloud Server, meeting the requirements of scenarios such as game acceleration, cross-border access, and global service deployment.

For operations, GA provides access log capabilities that deliver listener access request information to Log Tank Service (LTS) for unified storage and analysis, helping users troubleshoot access exceptions, analyze traffic characteristics, and meet security audit requirements, providing strong support for the continuous and stable operation of global acceleration services.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Global Accelerator (GA), helping you understand how to efficiently manage cloud GA resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for GA resources. These best practices will help you quickly get started with automated GA deployment and lay a solid foundation for subsequent global acceleration, access log management, and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy GA Access Log](ga_access_log.md) - Introduces how to use Terraform to automatically deploy a GA access log, including accelerator creation, listener configuration, LTS log group and log stream creation, and access log delivery.

## Reference Materials

- [Huawei Cloud Global Accelerator Product Documentation](https://support.huaweicloud.com/ga/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
