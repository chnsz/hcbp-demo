# Introduction

## What is Domain Name Service (DNS)

Domain Name Service (DNS) is a highly available, high-performance domain name resolution service provided by Huawei Cloud, supporting both public and private domain name resolution. DNS service provides intelligent resolution, load balancing, health check, and other functions, helping users achieve intelligent scheduling and failover of domain names.

DNS service supports multiple resolution types, including A records, AAAA records, CNAME records, MX records, etc., meeting domain name resolution requirements for different scenarios. Through DNS service, enterprises can implement high availability resolution, intelligent scheduling, failover, and other functions for domain names, improving application availability and user experience.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Domain Name Service (DNS), helping you understand how to efficiently manage cloud domain name resolution resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for DNS resources. These best practices will help you quickly get started with automated DNS deployment and lay a solid foundation for subsequent domain name resolution management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Custom Line](custom_line.md) - Introduces how to use Terraform to automatically deploy DNS custom lines, including line creation and IP address segment configuration.
* [Deploy Endpoint](endpoint.md) - Introduces how to use Terraform to automatically deploy DNS endpoints, including VPC creation, subnet configuration, and endpoint deployment.
* [Deploy Public Zone](public_zone.md) - Introduces how to use Terraform to automatically deploy DNS public zones, including domain creation, TTL configuration, DNSSEC settings, and router association.
* [Deploy Public Zone Cross Accounts](public_zone_cross_accounts.md) - Introduces how to use Terraform to automatically deploy cross-account public zone creation, including domain authorization, recordset creation, authorization verification, and subdomain zone creation.

## Reference Materials

- [Huawei Cloud DNS Product Documentation](https://support.huaweicloud.com/dns/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
