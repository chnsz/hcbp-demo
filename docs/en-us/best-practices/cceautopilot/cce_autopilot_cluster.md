# Deploy CCE Autopilot Cluster

## Application Scenario

Cloud Container Engine (CCE) Autopilot is a serverless Kubernetes cluster mode provided by Huawei Cloud, delivering O&M-free container orchestration for cloud-native applications. Autopilot clusters fully manage the control plane and node resources, so you do not need to create or manage nodes; you only need to submit workloads to run containerized applications. Autopilot clusters use the ENI container network mode and support Kubernetes community-native APIs and tools, making them suitable for cloud-native workloads that expect lower cluster O&M costs and pay-as-you-go billing.

This best practice will introduce how to use Terraform to automatically deploy a CCE Autopilot cluster, including VPC and subnet creation and CCE Autopilot cluster creation, helping you quickly get started with automated CCE Autopilot cluster deployment.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCE Autopilot Cluster (huaweicloud_cce_autopilot_cluster)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_cluster)

### Resource/Data Source Dependencies

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_cce_autopilot_cluster
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Virtual Private Cloud

Add the following script in the TF file (such as main.tf) to create a virtual private cloud:

```hcl
# Create a virtual private cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr, with a default value of 192.168.0.0/16

### 3. Create a Virtual Private Cloud Subnet

Add the following script in the TF file (such as main.tf) to create a virtual private cloud subnet:

```hcl
# Create a virtual private cloud subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing the ID of the virtual private cloud resource
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; when it is empty, the subnet CIDR is automatically calculated based on the VPC CIDR
- **gateway_ip**: Assigned by referencing the input variable gateway_ip; when it is empty, the gateway address is automatically calculated based on the subnet CIDR

### 4. Create a CCE Autopilot Cluster

Add the following script in the TF file (such as main.tf) to create a CCE Autopilot cluster:

```hcl
# Create a CCE Autopilot cluster resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "cluster_name" {
  description = "The CCE autopilot cluster name"
  type        = string
}

variable "cluster_alias" {
  description = "The alias of the cluster"
  type        = string
  default     = ""
}

variable "cluster_description" {
  description = "The description of the cluster"
  type        = string
  default     = ""
}

variable "cluster_category" {
  description = "The cluster type. Only Turbo is supported"
  type        = string
  default     = "Turbo"
}

variable "cluster_type" {
  description = "The master node architecture. Valid values are: VirtualMachine"
  type        = string
  default     = "VirtualMachine"
}

variable "cluster_annotations" {
  description = "The cluster annotations in the format of key-value pairs"
  type        = map(string)
  default     = {}
}

variable "cluster_custom_san" {
  description = "The custom SAN field in the API server certificate of the cluster"
  type        = list(string)
  default     = []
}

variable "cluster_eip_id" {
  description = "The EIP ID of the cluster"
  type        = string
  default     = ""
}

variable "cluster_enable_snat" {
  description = "Whether SNAT is configured for the cluster"
  type        = bool
  default     = false
}

variable "cluster_enable_swr_image_access" {
  description = "Whether SWR image access is enabled for the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_efs" {
  description = "Whether to delete the associated EFS when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_eni" {
  description = "Whether to delete the associated ENI when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_net" {
  description = "Whether to delete the associated network resources when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_obs" {
  description = "Whether to delete the associated OBS when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_sfs_turbo" {
  description = "Whether to delete the associated SFS Turbo when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_lts_reclaim_policy" {
  description = "The LTS reclaim policy. Valid values are: delete, retain"
  type        = string
  default     = "retain"
}

variable "cluster_service_network_cidr" {
  description = "The IPv4 CIDR of the service network"
  type        = string
  default     = ""
}

variable "cluster_enterprise_project_id" {
  description = "The ID of the enterprise project to which the cluster belongs"
  type        = string
  default     = "0"
}

variable "cluster_configurations_override" {
  description = "The component configuration items for override"
  type        = list(object({
    name  = string
    value = string
  }))
  default     = []
}

variable "cluster_configurations_override_name" {
  description = "The component name for configurations override"
  type        = string
  default     = ""
}

variable "cluster_tags" {
  description = "The tags of the cluster"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_cce_autopilot_cluster" "test" {
  name        = var.cluster_name
  flavor      = "cce.autopilot.cluster"
  alias       = var.cluster_alias
  description = var.cluster_description
  category    = var.cluster_category
  type        = var.cluster_type
  annotations = var.cluster_annotations
  custom_san  = var.cluster_custom_san
  eip_id      = var.cluster_eip_id

  enable_snat             = var.cluster_enable_snat
  enable_swr_image_access = var.cluster_enable_swr_image_access

  delete_efs   = var.cluster_delete_efs
  delete_eni   = var.cluster_delete_eni
  delete_net   = var.cluster_delete_net
  delete_obs   = var.cluster_delete_obs
  delete_sfs30 = var.cluster_delete_sfs_turbo

  lts_reclaim_policy = var.cluster_lts_reclaim_policy

  host_network {
    vpc    = huaweicloud_vpc.test.id
    subnet = huaweicloud_vpc_subnet.test.id
  }

  container_network {
    mode = "eni"
  }

  eni_network {
    subnets {
      subnet_id = huaweicloud_vpc_subnet.test.ipv4_subnet_id
    }
  }

  service_network {
    ipv4_cidr = var.cluster_service_network_cidr
  }

  extend_param {
    enterprise_project_id = var.cluster_enterprise_project_id
  }

  dynamic "configurations_override" {
    for_each = length(var.cluster_configurations_override) > 0 ? [1] : []

    content {
      name = var.cluster_configurations_override_name

      dynamic "configurations" {
        for_each = var.cluster_configurations_override

        content {
          name  = configurations.value.name
          value = configurations.value.value
        }
      }
    }
  }

  tags = var.cluster_tags

  lifecycle {
    ignore_changes = [
      annotations,
      tags["CCE-Cluster-ID"],
      service_network,
    ]
  }
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable cluster_name
- **flavor**: The cluster flavor, fixed to cce.autopilot.cluster
- **alias**: Assigned by referencing the input variable cluster_alias
- **description**: Assigned by referencing the input variable cluster_description
- **category**: Assigned by referencing the input variable cluster_category, with a default value of Turbo
- **type**: Assigned by referencing the input variable cluster_type, with a default value of VirtualMachine
- **annotations**: Assigned by referencing the input variable cluster_annotations
- **custom_san**: Assigned by referencing the input variable cluster_custom_san
- **eip_id**: Assigned by referencing the input variable cluster_eip_id
- **enable_snat**: Assigned by referencing the input variable cluster_enable_snat
- **enable_swr_image_access**: Assigned by referencing the input variable cluster_enable_swr_image_access
- **delete_efs**: Assigned by referencing the input variable cluster_delete_efs
- **delete_eni**: Assigned by referencing the input variable cluster_delete_eni
- **delete_net**: Assigned by referencing the input variable cluster_delete_net
- **delete_obs**: Assigned by referencing the input variable cluster_delete_obs
- **delete_sfs30**: Assigned by referencing the input variable cluster_delete_sfs_turbo
- **lts_reclaim_policy**: Assigned by referencing the input variable cluster_lts_reclaim_policy
- **host_network**: The host network configuration of the cluster; vpc and subnet reference the IDs of the virtual private cloud and subnet resources respectively
- **container_network**: The container network configuration of the cluster, with mode fixed to eni
- **eni_network**: The ENI network configuration of the cluster; subnet_id references the IPv4 subnet ID of the subnet resource
- **service_network**: The service network configuration of the cluster; ipv4_cidr is assigned by referencing the input variable cluster_service_network_cidr
- **extend_param**: The extended parameters of the cluster; enterprise_project_id is assigned by referencing the input variable cluster_enterprise_project_id
- **configurations_override**: The component configuration override items of the cluster, assigned by referencing the input variables cluster_configurations_override and cluster_configurations_override_name
- **tags**: Assigned by referencing the input variable cluster_tags

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through the `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Cluster variables
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
cluster_name        = "tf-test-autopilot-cluster"
cluster_description = "Created by terraform script"
cluster_tags = {
  owner = "terraform"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CCE Autopilot cluster
4. Run `terraform show` to view the created CCE Autopilot cluster

## Reference Information

- [Huawei Cloud Cloud Container Engine Product Documentation](https://support.huaweicloud.com/cce/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CCE Autopilot Cluster](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cceautopilot/cce-autopilot-cluster)
