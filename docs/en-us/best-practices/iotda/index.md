# Introduction

## What is IoT Device Access (IoTDA)

IoT Device Access (IoTDA) is an IoT platform service provided by Huawei Cloud, offering secure and reliable connectivity, data collection, and command delivery capabilities for massive devices. IoTDA supports multiple access protocols and network modes, helping users quickly build IoT applications and enabling bidirectional communication between devices and the cloud.

IoTDA provides core capabilities such as device management, product models, rule engine, and data forwarding. Through the rule engine, users can forward device-reported data to other Huawei Cloud services (such as OBS, DIS, and FunctionGraph) as needed for data storage, analysis, and processing. IoTDA also provides data flow control policies and data backlog policies to limit the tenant-level data forwarding TPS and control the backlog size and backlog time of forwarded data, ensuring the stability of the data forwarding link.

With IoTDA, enterprises can quickly complete device access and data onboarding to the cloud without building an IoT platform themselves, reducing the development and O&M costs of IoT applications and laying a solid foundation for subsequent device management, data analysis, and business innovation.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud IoT Device Access (IoTDA), helping you understand how to efficiently manage cloud IoTDA resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for IoTDA resources. These best practices will help you quickly get started with automated IoTDA deployment and lay a solid foundation for subsequent IoTDA management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy Data Flow Control and Backlog Policies](data_processing_policies.md) - Introduces how to use Terraform to automatically deploy data flow control and backlog policies, including data flow control policy creation, data backlog policy creation, and input parameter configuration.

## Reference Materials

- [Huawei Cloud IoTDA Product Documentation](https://support.huaweicloud.com/iotda/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
