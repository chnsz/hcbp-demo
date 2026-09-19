# Introduction

## What is Data Warehouse Service (DWS)

Data Warehouse Service (DWS) is an online analytical processing (OLAP) database service provided by Huawei Cloud. Built on Huawei Cloud infrastructure, it is designed for enterprise-level data warehouse scenarios and delivers fast query and analysis capabilities for massive data. DWS adopts a distributed architecture and columnar storage technology, enabling efficient processing of petabyte-scale data and helping enterprises quickly build high-performance, highly reliable data analysis platforms on the cloud.

DWS offers multiple cluster forms and specifications, supporting scenarios such as standard data warehouses, real-time data warehouses, and IoT data warehouses, and is compatible with the PostgreSQL ecosystem for smooth business migration. DWS provides enterprise-level capabilities such as elastic scaling, automatic backup, monitoring and alarming, and disaster recovery, meeting the diverse needs of data analysis services ranging from small and medium scale to ultra-large scale.

For O&M, DWS provides capabilities such as cluster event subscription, alarm notification, and performance monitoring, helping O&M personnel perceive key cluster events such as scaling, restart, and failures in time, so as to respond quickly and ensure business continuity and stability.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Warehouse Service (DWS), helping you understand how to efficiently manage cloud DWS resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DWS resources. These best practices will help you quickly get started with automated DWS deployment and lay a solid foundation for subsequent DWS cluster and event subscription management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Event Subscription](event_subscription.md) - Introduces how to use Terraform to automatically deploy an event subscription, including VPC creation, subnet and security group configuration, DWS cluster creation, SMN topic and subscription creation, and event subscription configuration.

## Reference Materials

- [Huawei Cloud Data Warehouse Service Product Documentation](https://support.huaweicloud.com/dws/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
