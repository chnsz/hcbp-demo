# Introduction

## What is Cloud Container Engine (CCE)

Cloud Container Engine (CCE) is a high-reliability, high-performance enterprise-grade container management service that supports Kubernetes community native applications and tools. It provides multiple types of container clusters, making it convenient for users to deploy, manage, and scale containerized applications. Through CCE, you can easily build Kubernetes-based containerized applications and implement microservices architecture.

CCE supports two cluster types: Standard and Turbo, providing full lifecycle management, supporting multiple types of nodes including ECS and BMS, and providing elastic scaling capabilities. It supports VPC networks and container tunnel networks, enabling efficient communication between containers, and provides multiple storage types to meet different application scenario requirements.

## Best Practices Overview

This section provides best practice examples for using Terraform to automatically deploy and manage Huawei Cloud Cloud Container Engine (CCE), helping you understand how to efficiently manage cloud container resources using Infrastructure as Code (IaC).

Through the best practices in this section, you can learn the main deployment processes for container cluster resources. These best practices will help you quickly get started with automated Cloud Container Engine deployment and lay a solid foundation for subsequent CCE management and operation work.

## Best Practices List

This section contains the following best practices:

* [Deploy AutoScaler Addon](addon_autoscaler.md) - Introduces how to use Terraform to automatically deploy a CCE AutoScaler addon, including querying CCE clusters, addon templates, and IAM projects, as well as creating CCE addons.
* [Deploy CoreDNS Addon](addon_coredns.md) - Introduces how to use Terraform to automatically deploy a CCE CoreDNS addon, including querying CCE clusters and addon templates, as well as creating CCE addons.
* [Deploy Kubernetes and Authenticate with Config](kubernetes_authentication_with_config.md) - Introduces how to use Terraform to automatically configure the Kubernetes provider by saving the KubeConfig configuration of the CCE cluster to a local file, and then using that configuration file to configure the Kubernetes provider to achieve connection to the CCE cluster.
* [Deploy Kubernetes Namespace](kubernetes_namespace.md) - Introduces how to use Terraform to automatically deploy a Kubernetes namespace, including querying availability zones and instance flavors, as well as creating infrastructure such as VPC, subnet, Elastic IP, CCE cluster, node, and Kubernetes namespace.
* [Deploy Kubernetes PVC using Existing OBS](kubernetes_pvc_with_existing_obs.md) - Introduces how to use Terraform to automatically deploy a complete solution for managing PVC with OBS, including querying availability zones and instance flavors, as well as creating infrastructure such as VPC, subnet, Elastic IP, CCE cluster, node, OBS bucket, and Kubernetes Secret, Persistent Volume, Persistent Volume Claim, and Deployment.
* [Deploy Kubernetes PVC using New OBS](kubernetes_pvc_with_new_obs.md) - Introduces how to use Terraform to automatically deploy a complete solution for managing PVC with new OBS, including querying availability zones and instance flavors, as well as creating infrastructure such as VPC, subnet, Elastic IP, CCE cluster, node, and Kubernetes Secret, Persistent Volume Claim, and Deployment. Unlike using existing OBS buckets, this best practice automatically creates OBS buckets and Persistent Volumes through PVC, simplifying the deployment process.
* [Deploy Node](node.md) - Introduces how to use Terraform to automatically deploy a CCE node, including querying availability zones and instance flavors, as well as creating VPC, subnet, Elastic IP, CCE cluster, key pair, and node.
* [Deploy Node Partition](node_partition.md) - Introduces how to use Terraform to automatically deploy a CCE node partition, including querying availability zones and instance flavors, as well as creating VPC, subnet, ENI subnet, CCE cluster, node partition, node, and node pool.
* [Deploy Node Pool](node_pool.md) - Introduces how to use Terraform to automatically deploy a CCE node pool, including querying availability zones and instance flavors, as well as creating VPC, subnet, Elastic IP, CCE cluster, key pair, and node pool.
* [Deploy Standard Cluster](standard_cluster.md) - Introduces how to use Terraform to automatically deploy a CCE Standard cluster, including querying availability zones, as well as creating VPC, subnet, Elastic IP, and CCE cluster.
* [Deploy Turbo Cluster](turbo_cluster.md) - Introduces how to use Terraform to automatically deploy a CCE Turbo cluster, including querying availability zones, as well as creating VPC, subnet, ENI subnet, Elastic IP, and CCE cluster.

## Reference Materials

- [Huawei Cloud CCE Product Documentation](https://support.huaweicloud.com/cce/index.html)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
