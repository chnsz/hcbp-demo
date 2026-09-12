# Introduction

## What is Data Replication Service (DRS)

Data Replication Service (DRS) is a one-stop data replication service provided by Huawei Cloud, dedicated to helping users achieve real-time synchronization, migration, and disaster recovery of databases. DRS supports data flow between multiple mainstream database engines, including relational databases and document databases, and can complete smooth data migration without stopping services, ensuring business continuity and data consistency.

DRS provides multiple capabilities such as real-time migration, real-time synchronization, data subscription, and backup migration, covering typical scenarios such as cloud migration, cross-region disaster recovery, active-active disaster recovery, and data backflow. Through the graphical task management interface and flexible migration policy configuration, users can easily define source and target connections, select migration objects, set filter rules, and monitor migration progress and data consistency in real time.

At the O&M level, DRS provides capabilities such as task monitoring, alarm notification, data comparison, and repair, helping users discover and handle exceptions during migration in a timely manner, reducing the O&M complexity of data migration and synchronization, and providing strong guarantees for the stable operation of services.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Replication Service (DRS), helping you understand how to efficiently manage cloud DRS resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DRS resources. These best practices will help you quickly get started with automated DRS deployment and lay a solid foundation for subsequent data replication, migration task management, and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy MongoDB Sharding Connection](drs_connection_mongodb.md) - Introduces how to use Terraform to automatically deploy a MongoDB sharding connection, including connection basic information configuration, primary node access information, shard node access information, SSL connection mode, and driver name configuration.

## Reference Materials

- [Huawei Cloud Data Replication Service Product Documentation](https://support.huaweicloud.com/drs/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
