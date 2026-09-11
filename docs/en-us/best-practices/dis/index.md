# Introduction

## What is Data Ingestion Service (DIS)

Data Ingestion Service (DIS) is a real-time data ingestion service provided by Huawei Cloud, used to collect and transmit massive amounts of data to the cloud in real time. DIS provides fully managed stream capabilities, supports the ingestion of various data sources and data types, and helps users build real-time data pipelines from data collection and transmission to processing and analysis, widely used in scenarios such as log collection, IoT data ingestion, real-time monitoring, and streaming analytics.

DIS uses the stream as its core concept. A stream is responsible for carrying data writes and reads, and can meet different throughput and reliability requirements through parameters such as the number of partitions and the data retention period. DIS supports both normal and advanced stream types, supports auto scaling of partitions, and supports multiple data types such as CSV, JSON, and BLOB as well as compression formats such as zip, gzip, snappy, lz4, and zstd, helping users flexibly adapt to different data formats and business scales.

With DIS, users can create and manage data ingestion streams on demand, and combine tag management to achieve resource classification and cost analysis, reducing the development and O&M costs of data ingestion and laying a solid foundation for subsequent real-time data processing and analysis.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Ingestion Service (DIS), helping you understand how to efficiently manage cloud DIS resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DIS resources. These best practices will help you quickly get started with automated DIS deployment and lay a solid foundation for subsequent DIS stream management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy DIS Stream](stream.md) - Introduces how to use Terraform to automatically deploy a DIS stream, including basic stream configuration, auto scaling partition configuration, data format and compression configuration, and tag management.

## Reference Materials

- [Huawei Cloud Data Ingestion Service Product Documentation](https://support.huaweicloud.com/dis/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
