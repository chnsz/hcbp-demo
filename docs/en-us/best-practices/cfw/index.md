# Introduction

## What is Cloud Firewall (CFW)

Cloud Firewall (CFW) is a new-generation cloud-native firewall that provides protection for internet boundaries and VPC boundaries on the cloud, including real-time intrusion detection and prevention, global unified access control, full traffic analysis visualization, log auditing and traceability analysis. It also supports on-demand elastic scaling, AI-enhanced intelligent defense, and flexible expansion to meet changing and expanding cloud business requirements, enabling users to quickly and flexibly respond to threats.

CFW supports access control policies based on five-tuple, domain names, applications, and IPS rules. It provides fine-grained control over north-south and east-west traffic at internet and VPC boundaries, integrates Huawei Cloud threat intelligence and AI intrusion prevention engines, and provides fundamental network security protection for cloud workloads.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Cloud Firewall (CFW), helping you understand how to efficiently manage CFW resources on the cloud using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for CFW resources. These best practices will help you quickly get started with automated CFW deployment and lay a solid foundation for subsequent firewall policy management and operations work.

## Best Practices List

This section contains the following best practices:

* [Deploy ACL Rule Configuration](acl_rule_config.md) - Introduces how to use Terraform to automatically deploy CFW ACL rule configuration, including querying firewall information and creating IP address groups, service groups, domain name groups, and multiple types of ACL access control rules.
* [Deploy Basic Firewall](basic_firewall.md) - Introduces how to use Terraform to automatically deploy a basic CFW firewall, including purchasing a firewall instance, enabling EIP auto-protection, and optionally manually binding existing EIPs for protection.
* [Deploy Black and White List](black_white_list.md) - Introduces how to use Terraform to automatically deploy CFW black and white list rules, including querying existing firewall information and creating blacklist and whitelist rules to block and allow specified IP addresses respectively.

## Reference Materials

- [Huawei Cloud CFW Product Documentation](https://support.huaweicloud.com/cfw/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
