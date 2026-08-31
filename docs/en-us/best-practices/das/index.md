# Introduction

## What is Data Admin Service (DAS)

Data Admin Service (DAS) is a Web service for logging in to and operating databases on Huawei Cloud. It provides a one-stop cloud database management platform for database development, O&M, and intelligent diagnosis, making it easier to use and maintain databases.

DAS provides visualized database operations, including basic SQL operations, advanced database management, and intelligent O&M, helping users manage databases in an easy-to-use, secure, and intelligent way. Without installing a local client, you can visually manage and maintain various database instances such as RDS, GaussDB, and DDS.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Admin Service (DAS), helping you understand how to efficiently manage DAS resources on the cloud using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DAS resources. These best practices will help you quickly get started with automated DAS deployment and lay a solid foundation for subsequent management and operations of database connections, lock analysis, database users, and shared connections.

## Best Practices List

This section contains the following best practices:

* [Deploy Database Connection](database_connection.md) - Introduces how to use Terraform to automatically deploy a DAS database instance connection, including creating a database instance connection, a database user, and sharing the connection with another IAM user.
* [Deploy Lock Analysis](lock_analysis.md) - Introduces how to use Terraform to automatically deploy DAS lock analysis configurations, including enabling full deadlock detection, enabling the historical transaction switch, and creating a historical transaction export task.

## Reference Materials

- [Huawei Cloud DAS Product Documentation](https://support.huaweicloud.com/das/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
