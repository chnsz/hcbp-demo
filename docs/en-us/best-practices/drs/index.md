# Introduction

## What is Data Replication Service (DRS)

Data Replication Service (DRS) is a one-stop data replication service provided by Huawei Cloud, dedicated to solving data flow problems in scenarios such as database cloud migration, database migration, real-time database synchronization, and database disaster recovery. DRS supports multiple mainstream database engines, including MySQL, PostgreSQL, SQL Server, MongoDB, and Oracle, and can achieve efficient data replication between on-cloud, off-cloud, and cross-cloud environments while ensuring business continuity.

DRS provides three core capabilities: real-time migration, real-time synchronization, and real-time disaster recovery. Real-time migration is used to smoothly migrate databases from on-premises or other clouds to Huawei Cloud, supporting online migration and resumable transfer. Real-time synchronization is used to achieve continuous data synchronization between databases, suitable for scenarios such as active-active business, read/write splitting, and data aggregation. Real-time disaster recovery is used to build cross-AZ or cross-region database disaster recovery capabilities, ensuring high availability of business.

Before using DRS to carry out data replication tasks, you need to maintain the access information of source and target databases through the connection management function, including database type, IP address and port, database username and password, SSL configuration, and driver configuration. A DRS connection is the foundation of subsequent migration, synchronization, and disaster recovery tasks. Terraform can be used to automatically create and manage these connections, improving deployment efficiency and reducing the risk of manual configuration errors.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Replication Service (DRS), helping you understand how to efficiently manage cloud DRS resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DRS resources. These best practices will help you quickly get started with automated DRS deployment and lay a solid foundation for subsequent DRS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy MongoDB Sharding Connection](drs_connection_mongodb.md) - Introduces how to use Terraform to automatically deploy a MongoDB sharding connection, including DRS connection creation, primary node access information configuration, shard node access information configuration, SSL configuration, and driver configuration.

## Reference Materials

- [Huawei Cloud Data Replication Service Product Documentation](https://support.huaweicloud.com/drs/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
