# Introduction

## What is DataArts Studio

DataArts Studio is a one-stop data operations and governance platform provided by Huawei Cloud, offering full lifecycle data management and intelligent data management capabilities, with support for intelligent industry knowledge base construction. DataArts Studio supports integration with Huawei Cloud data lakes and database cloud services, helping enterprises quickly build end-to-end intelligent data systems from data ingestion to data analysis, eliminating data silos, unifying data standards, and accelerating data monetization.

DataArts Studio provides functional modules such as data integration, data development, data architecture, data quality, data catalog, data service, and data security. DataArts Factory (data development) provides a fully managed big data scheduling and development environment, supporting script types such as DLI SQL for data management, job development, and operation monitoring.

## Best Practices Overview

This chapter provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud DataArts Studio, helping you understand how to efficiently manage cloud DataArts resources using Infrastructure as Code (IaC).

Through the best practices in this chapter, you can learn the deployment workflows of major DataArts Studio resources and associated DLI resources. These best practices will help you quickly get started with automated DataArts deployment and lay a solid foundation for subsequent data development script, job execution, and operation management work.

## Best Practice List

This chapter includes the following best practices:

* [Deploy DataArts Factory Script Execute](factory_script_execute.md) - Introduces how to use Terraform to automatically deploy DataArts Factory scripts and script execution, including workspace query, DLI data connection query, DLI database and table, Factory script, and script execution creation.
* [Deploy DataArts Studio Workspace User](studio_workspace_user.md) - Introduces how to use Terraform to automatically deploy DataArts Studio workspace users, including workspace query, user role query, IAM user query, and workspace user creation with role assignment.

## References

- [Huawei Cloud DataArts Studio Product Documentation](https://support.huaweicloud.com/intl/en-us/dataartsstudio/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
