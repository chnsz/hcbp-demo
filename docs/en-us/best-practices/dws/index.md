# Introduction

## What is Data Warehouse Service (DWS)

Data Warehouse Service (DWS) is an enterprise-grade cloud data warehouse service provided by Huawei Cloud. Built on Huawei Cloud infrastructure, it is designed for massive data analysis scenarios and delivers high-performance, highly reliable, and elastically scalable data warehouse capabilities. DWS adopts a distributed architecture and columnar storage technology, enabling efficient processing of complex queries and analysis over petabyte-scale data, helping enterprises quickly build data warehouses, data marts, and real-time analytics services.

DWS provides multiple cluster forms and specifications, supports various deployment modes such as standard data warehouse, real-time data warehouse, and cloud data warehouse, and is compatible with standard SQL and mainstream database ecosystem tools, making it easy for users to smoothly migrate existing data warehouse workloads. In addition, DWS offers enterprise-grade O&M capabilities such as automatic backup, monitoring and alarming, elastic scaling, and event subscription, helping users reduce O&M costs while ensuring business continuity.

With DWS, enterprises can centrally store and analyze scattered business data, unlock data value, and support business decisions and innovation, making it an important foundational service for building cloud data platforms.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Warehouse Service (DWS), helping you understand how to efficiently manage cloud DWS resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DWS resources. These best practices will help you quickly get started with automated DWS deployment and lay a solid foundation for subsequent DWS cluster and event subscription management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Event Subscription](event_subscription.md) - Introduces how to use Terraform to automatically deploy an event subscription, including VPC creation, subnet and security group configuration, DWS cluster creation, SMN topic and subscription configuration, and event subscription management.

## Reference Materials

- [Huawei Cloud Data Warehouse Service Product Documentation](https://support.huaweicloud.com/dws/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
