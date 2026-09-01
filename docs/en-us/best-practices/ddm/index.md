# Introduction

## What is Distributed Database Middleware (DDM)

Distributed Database Middleware (DDM) is a distributed relational database middleware. It is compatible with the MySQL protocol and focuses on solving database distributed scaling issues, breaking through capacity and performance bottlenecks of traditional databases to enable highly concurrent access to massive volumes of data.

Developed by Huawei Cloud, DDM is a cloud-native distributed database middleware that uses an architecture with decoupled storage and compute. It provides capabilities such as database and table sharding, read/write splitting, and elastic scaling, and features stability, reliability, high scalability, and continuous maintainability. Management of instance nodes is transparent to users. You can perform O&M and read/write operations on the DDM console, similar to using a traditional single-node database.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Distributed Database Middleware (DDM), helping you understand how to efficiently manage DDM resources on the cloud using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DDM resources. These best practices will help you quickly get started with automated DDM deployment and lay a solid foundation for subsequent DDM instance management and operations work.

## Best Practices List

This section contains the following best practices:

* [Deploy Account](account.md) - Introduces how to use Terraform to automatically deploy a DDM instance and account, including creating VPC, subnet, and security group, configuring engine, flavor, and node number, and creating a DDM account with specified permissions.
* [Deploy Basic Instance](basic_instance.md) - Introduces how to use Terraform to automatically deploy a basic DDM instance, including creating VPC, subnet, and security group, as well as configuring engine, flavor, node number, billing mode, and other parameters.

## Reference Materials

- [Huawei Cloud DDM Product Documentation](https://support.huaweicloud.com/ddm/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
