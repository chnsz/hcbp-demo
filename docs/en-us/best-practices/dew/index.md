# Introduction

## What is Data Encryption Workshop (DEW)

Data Encryption Workshop (DEW) is a data security service provided by Huawei Cloud, used to protect the security of cloud data and applications. DEW service provides key management, credential management, data encryption and other functions, supports multiple encryption algorithms and key types, helping users achieve encrypted storage and transmission of data, ensuring data security.

DEW service provides core functions such as CSMS (Cloud Secret Management Service), KMS (Key Management Service), etc., supports full lifecycle management of keys including creation, rotation, deletion, etc., with high availability and high reliability. Through DEW service, enterprises can securely manage sensitive information, achieve unified management and secure access to secrets, and improve application security.

## Best Practice Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Encryption Workshop (DEW) resources, helping you understand how to efficiently manage DEW resources on the cloud using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the deployment process of main DEW resources. These best practices will help you quickly get started with DEW automated deployment and lay a solid foundation for subsequent DEW management and operation work.

## Best Practice List

This section contains the following best practices:

* [Deploy CSMS Secret](csms_secret.md) - Introduces how to use Terraform to automatically create CSMS secrets, securely store and manage sensitive information such as passwords, API keys, certificates, etc., achieving unified management and secure access to secrets.
* [Deploy KMS Key](kms_key.md) - Introduces how to use Terraform to automatically create KMS keys, generate and manage keys for data encryption, support multiple encryption algorithms and key types, achieve encrypted storage and transmission of data, and ensure data security.

## Reference Materials

- [Huawei Cloud DEW Product Documentation](https://support.huaweicloud.com/dew/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
