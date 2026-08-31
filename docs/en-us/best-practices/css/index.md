# Introduction

## What is Cloud Search Service (CSS)

Cloud Search Service (CSS) is a collection of software in the Huawei Cloud ELK ecosystem. It is a fully managed online distributed search service based on Elasticsearch and OpenSearch, providing richer and more flexible retrieval and analysis capabilities. It can handle multi-condition retrieval, statistics, and reporting for structured and unstructured text as well as AI vector-based data.

CSS is compatible with Elasticsearch, OpenSearch, Logstash, Kibana, OpenSearch Dashboards, Cerebro, and other software. It supports automatic deployment and rapid cluster creation, with maintenance-free operation and a comprehensive monitoring system. It is suitable for scenarios such as log analysis, intelligent customer service, knowledge base Q&A, and personalized recommendations.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Cloud Search Service (CSS), helping you understand how to efficiently manage CSS resources on the cloud using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for CSS resources. These best practices will help you quickly get started with automated CSS deployment and lay a solid foundation for subsequent search cluster management and operations work.

## Best Practices List

This section contains the following best practices:

* [Deploy AI Ops Setting](ai_ops_setting.md) - Introduces how to use Terraform to automatically deploy CSS AI Ops settings, including creating VPC, subnet, security group, CSS cluster, and configuring AI Ops inspection settings.
* [Deploy Cluster](cluster.md) - Introduces how to use Terraform to automatically deploy a CSS cluster, including creating VPC, subnet, security group, and configuring cluster engine type, security mode, HTTPS access, and billing mode.

## Reference Materials

- [Huawei Cloud CSS Product Documentation](https://support.huaweicloud.com/css/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
