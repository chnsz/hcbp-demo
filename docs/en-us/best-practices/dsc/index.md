# Introduction

## What is Data Security Center (DSC)

Data Security Center (DSC) is a one-stop data security governance service provided by Huawei Cloud. It delivers capabilities such as sensitive data identification, data masking, data watermarking, and data security auditing across the full data lifecycle, helping enterprises build a visible, controllable, and auditable data security protection system that meets compliance requirements.

DSC supports automated sensitive data scanning, classification, and grading for various data sources such as cloud databases, big data services, and object storage, helping users quickly discover the distribution of sensitive data and identify potential risks. On this basis, DSC provides static and dynamic masking capabilities with built-in and custom algorithms, enabling masking of sensitive fields such as ID card numbers, mobile phone numbers, and bank card numbers to ensure data security in development, testing, and sharing scenarios.

With DSC, enterprises can uniformly manage data security policies, track data flow paths, and generate compliance audit reports, thereby reducing the risk of data leakage, improving data security governance efficiency, and laying a solid foundation for subsequent data security operations and compliance checks.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Data Security Center (DSC), helping you understand how to efficiently manage cloud DSC resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DSC resources. These best practices will help you quickly get started with automated DSC deployment and lay a solid foundation for subsequent data security and data masking management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Custom Character Mask Algorithm](custom_character_mask_algorithm.md) - Introduces how to use Terraform to automatically deploy a custom character mask algorithm, including mask algorithm name configuration, prefix and suffix character retention settings, and replacement character configuration.
* [Deploy Custom Scan Security Level](custom_scan_security_level.md) - Introduces how to use Terraform to automatically deploy a custom scan security level, including security level name configuration, console color number setting, and security level description configuration.
* [Deploy OBS Asset with Authorization](obs_asset_with_authorization.md) - Introduces how to use Terraform to automatically deploy OBS Asset with Authorization, including OBS Bucket, DSC Asset Authorization, and DSC OBS Asset.

## Reference Materials

- [Huawei Cloud Data Security Center Product Documentation](https://support.huaweicloud.com/dsc/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
