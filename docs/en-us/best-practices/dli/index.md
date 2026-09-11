# Introduction

## What is Data Lake Insight (DLI)

Data Lake Insight (DLI) is a big data computing and analysis service provided by Huawei Cloud, supporting multiple computing engines such as SQL, Flink, and Spark, helping users quickly build data lake analysis businesses. DLI provides fully managed computing resources, allowing users to process massive data easily without worrying about the operation and maintenance of underlying infrastructure, achieving data insights and business innovation.

DLI supports access to multiple data sources, including Object Storage Service (OBS), Data Ingestion Service (DIS), and relational databases, and provides a rich variety of job types, such as SQL jobs, Flink jobs, and Spark jobs, meeting the needs of batch processing, stream processing, and interactive query scenarios. Meanwhile, DLI provides elastic resource pool and queue management capabilities, allowing users to flexibly adjust computing resources based on business load, achieving a balance between cost and performance.

DLI also provides comprehensive data security and permission management functions, supporting fine-grained access control and data encryption to ensure the security and compliance of enterprise data. With DLI, enterprises can quickly build a data lake analysis platform, mine data value, and drive business decisions.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Lake Insight (DLI), helping you understand how to efficiently manage cloud DLI resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DLI resources. These best practices will help you quickly get started with automated DLI deployment and lay a solid foundation for subsequent management and operation work of elastic resource pools, queues, and Flink jobs.

## Best Practices List

This section contains the following best practices:

* [Deploy Flink Jar Job](flink_jar_job.md) - Introduces how to use Terraform to automatically deploy a Flink Jar job, including elastic resource pool creation, general queue configuration, and job parameter settings.
* [Deploy Flink OpenSource SQL Job](flink_opensource_sql_job.md) - Introduces how to use Terraform to automatically deploy a Flink OpenSource SQL job, including elastic resource pool creation, general queue configuration, and job parameter settings.
* [Deploy Queue Public Network Connectivity](queue_public_network_connectivity.md) - Introduces how to use Terraform to automatically deploy DLI queue public network connectivity, including elastic resource pool and queue creation, VPC and subnet configuration, enhanced datasource connection association, EIP and NAT gateway creation, and SNAT rule configuration.

## Reference Materials

- [Huawei Cloud Data Lake Insight Product Documentation](https://support.huaweicloud.com/dli/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
