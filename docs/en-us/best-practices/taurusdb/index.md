# Introduction

## What is TaurusDB

TaurusDB is an enterprise-grade cloud-native database service provided by Huawei Cloud, fully compatible with the MySQL protocol and built on a compute-storage separation architecture. By decoupling compute nodes from storage nodes, TaurusDB supports on-demand storage scaling and second-level elastic scaling of compute nodes, enabling it to handle fluctuating business workloads and helping enterprises build high-performance, highly reliable database services on the cloud.

TaurusDB provides one-writer-multiple-readers capability, allowing a single instance to attach multiple read replicas so that read requests can be automatically load balanced across them, significantly improving throughput in high-concurrency read scenarios. It also offers enterprise-grade capabilities such as parallel query, seconds-level monitoring, SQL audit, parameter templates, and backup and recovery, making it suitable for online transaction processing (OLTP) and other scenarios with high requirements on performance, reliability, and scalability.

With TaurusDB, enterprises can obtain a near-native MySQL experience with higher performance without building and maintaining database clusters themselves, allowing them to focus more on business development and innovation.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud TaurusDB, helping you understand how to efficiently manage cloud TaurusDB resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for TaurusDB resources. These best practices will help you quickly get started with automated TaurusDB deployment and lay a solid foundation for subsequent TaurusDB instance, account, and database management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy TaurusDB Instance](taurusdb_instance.md) - Introduces how to use Terraform to automatically deploy a TaurusDB instance, including VPC creation, subnet creation, security group configuration, parameter template, instance configuration, account and database management.

## Reference Materials

- [Huawei Cloud TaurusDB Product Documentation](https://support.huaweicloud.com/taurusdb/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
