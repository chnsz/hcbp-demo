# Introduction

## What is Content Delivery Network (CDN)

Content Delivery Network (CDN) is a content acceleration and distribution service provided by Huawei Cloud, which provides users with nearby access acceleration experience by caching content to globally distributed edge nodes. CDN service supports multiple scenarios such as static content acceleration, dynamic content acceleration, video on-demand and live streaming acceleration, helping users improve website access speed, reduce origin server pressure, and enhance user experience.

CDN service provides flexible cache policy configuration, HTTPS acceleration, anti-leech, access control and other security features, supports real-time monitoring and statistical analysis, and has high availability and high reliability. Through CDN service, enterprises can quickly build a global content delivery network, achieve rapid content distribution and access, reduce bandwidth costs, and improve business competitiveness.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Content Delivery Network (CDN), helping you understand how to efficiently manage cloud CDN resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for CDN resources. These best practices will help you quickly get started with automated CDN deployment and lay a solid foundation for subsequent CDN management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Cache Management](cache_management.md) - Introduces how to use Terraform to automatically execute CDN cache refresh and preheat operations, managing cached content on CDN nodes to ensure users get the latest resources and improve access speed.
* [Deploy HTTPS and Cache Domain](domain_with_https_and_cache.md) - Introduces how to use Terraform to automatically create a CDN domain, including HTTPS and cache rule configuration, achieving content acceleration and secure transmission.
* [Deploy Rule Engine](rule_engine.md) - Introduces how to use Terraform to automatically configure CDN rule engine rules, executing corresponding actions based on different request conditions, achieving fine-grained CDN acceleration control.

## Reference Information

- [Huawei Cloud CDN Product Documentation](https://support.huaweicloud.com/cdn/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
