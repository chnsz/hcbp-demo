# Introduction

## What is Image Management Service (IMS)

Image Management Service (IMS) is an image management service provided by Huawei Cloud, supporting image creation, sharing, copying, importing, exporting, and other functions. IMS service provides various image types including private images, shared images, and market images, supporting image creation from cloud servers, cloud disks, external image files, and other methods, meeting image management requirements for different scenarios.

IMS service supports cross-region and cross-account image sharing, providing complete image lifecycle management capabilities, supporting image version management, tag management, permission control, and other functions. Through IMS service, enterprises can achieve unified image management and sharing, improve the efficiency and convenience of image management, and ensure rapid deployment and migration of business systems.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Image Management Service (IMS), helping you understand how to efficiently manage cloud IMS resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for IMS resources. These best practices will help you quickly get started with automated IMS deployment and lay a solid foundation for subsequent image management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Cross Account Migration with Data Image](cross_account_migration_with_data_image.md) - Introduces how to use Terraform to automatically deploy cross-account migration with data images, including creating ECS instances, data disks, and data images in the sharer account, sharing images to the accepter account, accepting shared images in the accepter account, and creating data disks using shared images.
* [Deploy Cross Account Migration with Whole Image](cross_account_migration_with_whole_image.md) - Introduces how to use Terraform to automatically deploy cross-account migration with whole images, including creating ECS instances, CBR vaults, and whole images in the sharer account, sharing images to the accepter account, accepting shared images in the accepter account, and creating new ECS instances using shared images.

## Reference Materials

- [Huawei Cloud IMS Product Documentation](https://support.huaweicloud.com/ims/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
