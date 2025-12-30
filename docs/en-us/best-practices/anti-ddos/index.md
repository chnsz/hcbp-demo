# Introduction

## What is Anti-DDoS

Anti-DDoS (Anti-Distributed Denial of Service) is a distributed denial-of-service attack protection service provided by Huawei Cloud, which can effectively protect public IPs from DDoS attacks and ensure stable business operations. Anti-DDoS service provides two protection modes: Basic Protection and Professional Protection. Basic Protection provides free DDoS attack protection capabilities for Huawei Cloud users. When a DDoS attack is detected, the system will automatically start traffic cleaning, filter out attack traffic, and only forward normal traffic to the origin server.

Anti-DDoS service supports protection against multiple attack types, including common DDoS attack types such as SYN Flood, UDP Flood, ICMP Flood, HTTP Flood, etc. By configuring traffic cleaning thresholds and message notifications, users can timely understand the protection status and ensure safe and stable business operations.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Anti-DDoS, helping you understand how to efficiently manage cloud Anti-DDoS protection resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for Anti-DDoS protection resources. These best practices will help you quickly get started with automated Anti-DDoS deployment and lay a solid foundation for subsequent Anti-DDoS management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Anti-DDoS Basic Protection](basic.md) - Introduces how to use Terraform to automatically deploy Anti-DDoS Basic Protection, including creating Elastic IP, Simple Message Notification topics and subscriptions, and configuring Anti-DDoS Basic Protection.

## Reference Materials

- [Huawei Cloud Anti-DDoS Product Documentation](https://support.huaweicloud.com/antiddos/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
