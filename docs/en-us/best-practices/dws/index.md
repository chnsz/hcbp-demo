# Introduction

## What is Data Warehouse Service (DWS)

Data Warehouse Service (DWS) is an online analytical processing (OLAP) enterprise-level data warehouse service provided by Huawei Cloud. Built on Huawei Cloud infrastructure and designed for massive data analysis scenarios, DWS delivers high-performance, highly reliable, and easily scalable data warehouse capabilities. Adopting a distributed massively parallel processing (MPP) architecture with columnar storage and a vectorized execution engine, DWS enables fast query and analysis of trillions of records, helping enterprises quickly build data warehouse and business intelligence (BI) applications on the cloud.

DWS offers multiple forms such as standard data warehouse, real-time data warehouse, and cloud data warehouse. It is compatible with standard SQL and the PostgreSQL ecosystem, supports data import and export, data sharing, and hot/cold data tiered storage, and can work with services such as DataArts Studio, Object Storage Service (OBS), and Data Lake Insight (DLI) to build an end-to-end data analysis pipeline.

For operations, DWS provides enterprise-level capabilities such as cluster monitoring, event subscription, alarm notification, snapshot backup, and disaster recovery, helping O&M personnel promptly perceive cluster running status and abnormal events and ensuring the stability and continuity of the data warehouse service.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Warehouse Service (DWS), helping you understand how to efficiently manage cloud DWS resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DWS resources. These best practices will help you quickly get started with automated DWS deployment and lay a solid foundation for subsequent DWS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Event Subscription](event_subscription.md) - Introduces how to use Terraform to automatically deploy a DWS event subscription, including VPC creation, subnet and security group configuration, DWS cluster creation, SMN topic and subscription creation, and event subscription configuration.
* [Deploy Real-Time MySQL-to-DWS Data Synchronization](real_time_sync_mysql_to_dws.md) - Introduces how to use Terraform to automatically deploy the infrastructure for real-time MySQL-to-DWS data synchronization, including VPC, subnet, and security group creation, RDS MySQL instance creation, DWS cluster creation, DLI elastic resource pool and queue creation, and enhanced datasource connection association.

## Reference Materials

- [Huawei Cloud Data Warehouse Service Product Documentation](https://support.huaweicloud.com/dws/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
