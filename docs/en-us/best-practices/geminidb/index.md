# Introduction

## What is GeminiDB

GeminiDB is a multi-model NoSQL database service provided by Huawei Cloud that is compatible with multiple mainstream NoSQL engine protocols such as Cassandra, MongoDB, InfluxDB, and Redis. It offers high availability, high reliability, elastic scaling, and security, and can meet business requirements such as massive data storage, high-concurrency read and write, and diverse data models.

GeminiDB adopts a compute-storage separation architecture and a distributed cluster design, supporting independent elastic scaling of compute nodes and storage capacity, and provides enterprise-level capabilities such as automatic backup, failover, and monitoring and alarming, helping users quickly build stable and reliable NoSQL database services on the cloud.

With GeminiDB, users can select different engines and flavors on demand to flexibly handle data storage and access challenges in scenarios such as IoT, gaming, social networking, and financial risk control, reducing database O&M costs and improving business continuity and data security.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud GeminiDB, helping you understand how to efficiently manage cloud GeminiDB resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for GeminiDB resources. These best practices will help you quickly get started with automated GeminiDB deployment and lay a solid foundation for subsequent GeminiDB management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy GeminiDB Cassandra Instance](geminidb_cassandra_instance.md) - Introduces how to use Terraform to automatically deploy a GeminiDB Cassandra instance, including VPC creation, subnet creation, security group configuration, instance flavor querying, password generation, backup strategy, and instance backup management.

## Reference Materials

- [Huawei Cloud GeminiDB Product Documentation](https://support.huaweicloud.com/geminidb/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
