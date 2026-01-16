# Introduction

## What is Resource Access Manager (RAM)

Resource Access Manager (RAM) is a resource sharing service provided by Huawei Cloud, supporting cross-account resource sharing and management to help you achieve unified resource management and access control. RAM service provides centralized resource sharing capabilities, supporting cross-account resource sharing, permission management, and access control, meeting enterprise-level resource sharing and compliance requirements.

RAM service supports sharing of multiple resource types, including network resources such as VPCs, subnets, security groups, and route tables, as well as computing and storage resources such as ECS and RDS. Through RAM service, enterprises can achieve cross-account resource sharing, improve resource utilization, reduce management costs, while ensuring resource security and compliance.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Resource Access Manager (RAM), helping you understand how to efficiently manage RAM resources on the cloud using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the deployment processes of main RAM resources. These best practices will help you quickly get started with RAM automated deployment and lay a solid foundation for subsequent resource sharing, permission management, and access control operations.

## Best Practices List

This section contains the following best practices:

* [Deploy Automated Resource Share Invitation Processing](automated_resource_share_invitation_processing.md) - Introduces how to use Terraform to automatically process resource share invitations, including querying pending invitations and batch accepting or rejecting invitations.
* [Deploy Cross-Account Resource Share](cross_account_resource_share.md) - Introduces how to use Terraform to automatically deploy cross-account resource sharing, including resource share instance creation, principal configuration, resource URN configuration, and permission configuration.
* [Deploy Fine-Grained Permission Management](fine_grained_permission_management.md) - Introduces how to use Terraform to automatically deploy fine-grained permission management, including querying available permissions and associating permissions with resource shares.

## Reference Information

- [Huawei Cloud RAM Product Documentation](https://support.huaweicloud.com/ram/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
