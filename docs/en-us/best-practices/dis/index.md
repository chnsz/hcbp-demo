# Introduction

## What is Data Ingestion Service (DIS)

Data Ingestion Service (DIS) is a real-time data ingestion service provided by Huawei Cloud, offering fully managed, high-performance, and highly reliable data ingestion capabilities for users who process or analyze streaming data. DIS can be widely applied in scenarios such as real-time monitoring, log analysis, and IoT device data collection, helping users easily build applications based on streaming data.

DIS provides multiple data ingestion methods, supporting real-time collection of data from various data sources (such as applications, IoT devices, log files, etc.) and streaming the data to other services on Huawei Cloud (such as Object Storage Service OBS, Data Lake Insight DLI, MapReduce Service MRS, etc.) for further processing and analysis. DIS supports auto scaling, which can automatically adjust the number of partitions of the stream based on changes in business traffic, ensuring the stability and cost-effectiveness of data ingestion.

By using DIS, users can quickly build real-time data processing pipelines, realizing real-time collection, transmission, and distribution of data, providing a reliable data foundation for business scenarios such as real-time monitoring, real-time analysis, and alarm triggering.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Ingestion Service (DIS), helping you understand how to efficiently manage cloud DIS resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DIS resources. These best practices will help you quickly get started with automated DIS deployment and lay a solid foundation for subsequent DIS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy DIS Stream](stream.md) - Introduces how to use Terraform to automatically deploy a DIS stream, including auto scaling, data format, and compression format configuration.

## Reference Materials

- [Huawei Cloud Data Ingestion Service Product Documentation](https://support.huaweicloud.com/dis/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
