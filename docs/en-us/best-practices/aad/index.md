# Introduction

## What is Advanced Anti-DDoS (AAD)

Advanced Anti-DDoS (AAD) is a professional DDoS protection service provided by Huawei Cloud, designed to protect internet servers and applications from distributed denial-of-service (DDoS) attacks and other malicious traffic. AAD provides comprehensive protection capabilities, including DDoS traffic cleaning, CC (Challenge Collapsar) attack protection, and intelligent traffic analysis, ensuring the availability and stability of online services.

AAD supports multiple access modes, including website access and IP access, to meet the protection needs of different business scenarios. By directing business traffic to AAD cleaning nodes, AAD can detect and filter malicious traffic in real time, forwarding only legitimate traffic to the origin server, thereby effectively ensuring business continuity and security.

In addition, AAD offers flexible bandwidth configuration and protection policies, supporting on-demand adjustment of service bandwidth, elastic bandwidth, and protection packages, helping enterprises maintain business stability when facing sudden traffic attacks. The deployment and management of AAD can be automated using tools such as Terraform, further improving operational efficiency.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Advanced Anti-DDoS (AAD), helping you understand how to efficiently manage cloud AAD resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for AAD resources. These best practices will help you quickly get started with automated AAD deployment and lay a solid foundation for subsequent AAD management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Black/White Lists](black_white_lists.md) - Introduces how to use Terraform to create an AAD instance and configure black/white lists to block malicious IPs and allow trusted IPs.

## Reference Materials

- [Huawei Cloud Advanced Anti-DDoS Product Documentation](https://support.huaweicloud.com/aad/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
