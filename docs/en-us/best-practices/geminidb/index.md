# Introduction

## What is GeminiDB

GeminiDB is a cloud-native multi-model NoSQL database service provided by Huawei Cloud, built on a compute-storage separation architecture and compatible with multiple mainstream NoSQL interfaces such as Cassandra, MongoDB, InfluxDB, and Redis, meeting data storage and access requirements in various business scenarios. GeminiDB decouples compute nodes from storage nodes, supports on-demand storage scaling and second-level elastic scaling of compute nodes, helping enterprises maintain stable performance when business workloads fluctuate.

GeminiDB delivers high reliability, high availability, and high performance, ensuring data security and business continuity through mechanisms such as multi-replica redundancy, automatic failover, and backup and restoration. It also provides rich O&M capabilities, including monitoring and alarming, parameter templates, and backup management, helping users reduce database O&M costs and focus on business development and innovation.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud GeminiDB, helping you understand how to efficiently manage cloud GeminiDB resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for GeminiDB resources. These best practices will help you quickly get started with automated GeminiDB deployment and lay a solid foundation for subsequent GeminiDB management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy GeminiDB Cassandra Instance](geminidb_cassandra_instance.md) - Introduces how to use Terraform to automatically deploy a GeminiDB Cassandra instance, including VPC creation, subnet creation, security group configuration, flavor querying, random password generation, and instance backup management.

## Reference Materials

- [Huawei Cloud GeminiDB Product Documentation](https://support.huaweicloud.com/geminidb/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
