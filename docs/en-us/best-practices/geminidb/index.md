# Introduction

## What is GeminiDB

GeminiDB is a cloud-native multi-model NoSQL database service provided by Huawei Cloud, built on a compute-storage separation architecture and compatible with multiple mainstream NoSQL interfaces such as Cassandra, MongoDB, Redis, and InfluxDB. Designed for high-concurrency read and write scenarios with massive data volumes, GeminiDB delivers high availability, high reliability, and elastic scalability, helping enterprises build a modern data storage foundation with lower O&M costs.

GeminiDB adopts a compute-storage separation and distributed shared storage architecture, supporting second-level elastic scaling and on-demand scaling to effectively handle business workload fluctuations. It also provides enterprise-grade capabilities such as automatic backup, automatic failover, and monitoring and alarming, ensuring data security and business continuity.

With GeminiDB, you do not need to build and maintain NoSQL database clusters yourself. You can obtain a stable, high-performance multi-model database service and focus more on business development and innovation.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud GeminiDB, helping you understand how to efficiently manage cloud GeminiDB resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for GeminiDB resources. These best practices will help you quickly get started with automated GeminiDB deployment and lay a solid foundation for subsequent GeminiDB management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy GeminiDB Cassandra Instance](geminidb_cassandra_instance.md) - Introduces how to use Terraform to automatically deploy a GeminiDB Cassandra instance, including VPC and subnet creation, availability zone and flavor query, security group configuration, instance creation, and instance backup.

## Reference Materials

- [Huawei Cloud GeminiDB Product Documentation](https://support.huaweicloud.com/geminidb/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
