# Introduction

## What is Identity and Access Management (IAM)

Identity and Access Management (IAM) is a fundamental identity authentication and access management service provided by Huawei Cloud, offering core functions such as identity management, permission management, and access control for Huawei Cloud users. IAM service supports fine-grained permission control, helping enterprises achieve unified, secure, and efficient cloud resource access management through flexible combinations of users, user groups, roles, and policies.

IAM service provides a complete identity authentication and authorization system, supporting multiple authentication methods, Single Sign-On (SSO), Multi-Factor Authentication (MFA), and other security features, meeting enterprise-level identity management and compliance requirements. Through IAM service, enterprises can implement the principle of least privilege, precisely control user access permissions to cloud resources, and improve the security and management efficiency of cloud resources.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Identity and Access Management (IAM), helping you understand how to efficiently manage IAM resources on the cloud using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for IAM resources. These best practices will help you quickly get started with automated IAM deployment and lay a solid foundation for subsequent user, user group, role, and permission management operations.

## Best Practices List

This section contains the following best practices:

* [Deploy Group Policies Associate](group_policies_associate.md) - Introduces how to use Terraform to automatically deploy IAM user group and policy associations, including querying IAM policies, creating user groups, and associating policies with user groups.
* [Deploy Password Policy](password_policy.md) - Introduces how to use Terraform to automatically deploy IAM password policies, including configuration of security policies such as password length, character combination, validity period, reuse rules, etc.
* [Deploy Users Authorized Through Group](users_authorized_through_group.md) - Introduces how to use Terraform to automatically deploy IAM roles, user groups, and users, and authorize users through user groups, implementing group-based permission management.

## Reference Materials

- [Huawei Cloud IAM Product Documentation](https://support.huaweicloud.com/iam/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
