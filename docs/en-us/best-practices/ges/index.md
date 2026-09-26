# Introduction

## What is Graph Engine Service (GES)

Graph Engine Service (GES) is a one-stop graph data management and analysis service provided by Huawei Cloud, designed for scenarios such as social networks, knowledge graphs, financial risk control, and recommendation systems. It delivers graph data storage for billions of vertices and edges, millisecond-level query, and graph analysis capabilities. GES encapsulates graph storage, computing, and analysis as a cloud service, helping users quickly build graph applications without building and maintaining their own graph database clusters.

GES supports multiple graph specifications and CPU architectures, and provides full lifecycle management capabilities such as graph instance creation, backup, restoration, and import/export. It also supports HTTPS encrypted access and configurable data encryption algorithms, meeting business requirements of different scales and security levels. Users can flexibly manage graph instances through the console or APIs and adjust specifications on demand, lowering the barrier to building and operating graph applications.

With GES, enterprises can turn complex relationship networks into queryable and analyzable graph data assets, quickly uncover the value of relationships between data, and support services such as intelligent recommendation, anti-fraud, and knowledge reasoning.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Graph Engine Service (GES), helping you understand how to efficiently manage cloud GES resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for GES resources. These best practices will help you quickly get started with automated GES deployment and lay a solid foundation for subsequent GES graph instance and graph backup management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Graph Backup](backup.md) - Introduces how to use Terraform to automatically deploy a graph backup, including VPC creation, subnet configuration, security group creation, graph instance configuration, and graph backup creation.

## Reference Materials

- [Huawei Cloud Graph Engine Service Product Documentation](https://support.huaweicloud.com/ges/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
