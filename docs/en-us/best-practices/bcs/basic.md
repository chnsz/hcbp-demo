# Deploy Basic Instance

## Application Scenario

Blockchain Service (BCS) is a blockchain technology service platform for enterprises and developers. It helps you quickly deploy, manage, and maintain blockchain networks, lowering the barrier to using blockchain so that you can focus on business development and innovation and rapidly bring services on-chain. BCS supports creating Hyperledger Fabric enhanced edition and Huawei Cloud blockchain engine instances, and provides capabilities such as user management, node management, and O&M monitoring to meet enterprise-level and financial-level requirements.

This best practice introduces how to use Terraform to automatically deploy a basic BCS instance, including specifying the service edition, Fabric version, consensus algorithm, associating a CCE cluster, and configuring block generation parameters, SFS Turbo, peer organizations, and channels.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [BCS Instance Resource (huaweicloud_bcs_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/bcs_instance)

### Resource/Data Source Dependencies

```
huaweicloud_bcs_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Create BCS Instance

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a BCS instance resource:

```hcl
variable "instance_name" {
  description = "The unique name of the BCS instance"
  type        = string
}

variable "edition" {
  description = "The service edition of the BCS instance"
  type        = number
}

variable "fabric_version" {
  description = "The version of fabric for the BCS instance"
  type        = string
}

variable "consensus" {
  description = "The consensus algorithm used by the BCS instance"
  type        = string
}

variable "orderer_node_num" {
  description = "The number of peers in the orderer organization"
  type        = number
  default     = 1
}

variable "cce_cluster_id" {
  description = "The ID of the CCE cluster to attach to the BCS instance"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project that the BCS instance belongs to"
  type        = string
}

variable "instance_password" {
  description = "The resource access and blockchain management password"
  type        = string
  sensitive   = true
}

variable "volume_type" {
  description = "The storage volume type to attach to each organization of the BCS instance"
  type        = string
  default     = "nfs"
}

variable "org_disk_size" {
  description = "The storage capacity of peer organization"
  type        = number
  default     = 100
}

variable "block_info" {
  description = "The configuration of block generation"
  type = list(object({
    generation_interval  = optional(number, 2)
    transaction_quantity = optional(number, 500)
    block_size           = optional(number, 2)
  }))

  default = []
}

variable "sfs_turbo" {
  description = "The SFS Turbo configuration for BCS instance"
  type = list(object({
    share_type        = optional(string, "STANDARD")
    type              = optional(string, "efs-ha")
    availability_zone = optional(string, "")
    flavor            = optional(string, "sfs.turbo.20MBps")
  }))

  default = []

  validation {
    condition     = var.volume_type != "efs" || var.edition != 4 || length(var.sfs_turbo) > 0
    error_message = "When using \"volume_type = \"efs\"\" with \"edition = 4\", you must configure \"sfs_turbo\"."
  }
}

variable "peer_orgs" {
  description = "The array of one or more peer organizations to attach to the BCS instance"
  type = list(object({
    org_name = string
    count    = number
  }))

  default = [
    {
      org_name = "organization"
      count    = 2
    }
  ]
}

variable "channels" {
  description = "The array of one or more channels to attach to the BCS instance"
  type = list(object({
    name      = string
    org_names = list(string)
  }))

  default = [
    {
      name      = "channel"
      org_names = ["organization"]
    }
  ]
}

# Create BCS instance resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_bcs_instance" "test" {
  name                  = var.instance_name
  edition               = var.edition
  fabric_version        = var.fabric_version
  consensus             = var.consensus
  orderer_node_num      = var.orderer_node_num
  cce_cluster_id        = var.cce_cluster_id
  enterprise_project_id = var.enterprise_project_id
  password              = var.instance_password
  volume_type           = var.volume_type
  org_disk_size         = var.org_disk_size

  dynamic "block_info" {
    for_each = var.block_info

    content {
      generation_interval  = block_info.value.generation_interval
      transaction_quantity = block_info.value.transaction_quantity
      block_size           = block_info.value.block_size
    }
  }

  dynamic "sfs_turbo" {
    for_each = var.sfs_turbo

    content {
      share_type        = sfs_turbo.value.share_type
      type              = sfs_turbo.value.type
      availability_zone = sfs_turbo.value.availability_zone
      flavor            = sfs_turbo.value.flavor
    }
  }

  dynamic "peer_orgs" {
    for_each = var.peer_orgs

    content {
      org_name = peer_orgs.value.org_name
      count    = peer_orgs.value.count
    }
  }

  dynamic "channels" {
    for_each = var.channels

    content {
      name      = channels.value.name
      org_names = channels.value.org_names
    }
  }
}
```

**Parameter Description**:
- **name**: The unique name of the BCS instance, assigned by referencing the input variable instance_name
- **edition**: The service edition of the BCS instance, assigned by referencing the input variable edition, valid values are 1, 2, and 4
- **fabric_version**: The Fabric version used by the BCS instance, assigned by referencing the input variable fabric_version
- **consensus**: The consensus algorithm used by the BCS instance, assigned by referencing the input variable consensus
- **orderer_node_num**: The number of peers in the orderer organization, assigned by referencing the input variable orderer_node_num, defaults to 1
- **cce_cluster_id**: The ID of the CCE cluster attached to the BCS instance, assigned by referencing the input variable cce_cluster_id
- **enterprise_project_id**: The ID of the enterprise project that the BCS instance belongs to, assigned by referencing the input variable enterprise_project_id
- **password**: The resource access and blockchain management password, assigned by referencing the input variable instance_password
- **volume_type**: The storage volume type attached to each organization of the BCS instance, assigned by referencing the input variable volume_type, defaults to "nfs", valid values are "nfs" (SFS) and "efs" (SFS Turbo)
- **org_disk_size**: The storage capacity of the peer organization, assigned by referencing the input variable org_disk_size, defaults to 100
- **block_info**: The block generation configuration, created through the dynamic block `dynamic "block_info"` based on the input variable block_info
  - **generation_interval**: The block generation interval in seconds, defaults to 2
  - **transaction_quantity**: The number of transactions included in the block, defaults to 500
  - **block_size**: The block size in MB, defaults to 2
- **sfs_turbo**: The SFS Turbo file system configuration, created through the dynamic block `dynamic "sfs_turbo"` based on the input variable sfs_turbo
  - **share_type**: The share type of SFS Turbo, defaults to "STANDARD"
  - **type**: The type of SFS Turbo, defaults to "efs-ha"
  - **availability_zone**: The availability zone of SFS Turbo, defaults to an empty string
  - **flavor**: The flavor of SFS Turbo, defaults to "sfs.turbo.20MBps"
- **peer_orgs**: The peer organizations attached to the BCS instance, created through the dynamic block `dynamic "peer_orgs"` based on the input variable peer_orgs, defaults to creating one organization named organization with 2 peers
  - **org_name**: The name of the peer organization
  - **count**: The number of peers in the organization
- **channels**: The channels attached to the BCS instance, created through the dynamic block `dynamic "channels"` based on the input variable channels, defaults to creating one channel named channel
  - **name**: The name of the channel
  - **org_names**: The list of peer organization names associated with the channel

> Note: The BCS service needs to exclusively occupy the CCE cluster. Ensure that the target CCE cluster is not occupied by another BCS instance before deployment. When `volume_type` is "efs" and `edition` is 4, you must configure `sfs_turbo`.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# BCS instance configuration
instance_name         = "basic-bcs-instance"
edition               = 4
fabric_version        = "4.0.35"
consensus             = "etcdraft"
orderer_node_num      = 3
cce_cluster_id        = "your-cce-cluster-id"
enterprise_project_id = "your-enterprise-project-id"
instance_password     = "your-instance-password"
volume_type           = "efs"
org_disk_size         = 3686

block_info = [
  {
    generation_interval  = 2
    transaction_quantity = 500
    block_size           = 2
  }
]

# SFS Turbo configuration (required when volume_type is efs and edition is 4)
sfs_turbo = [
  {
    share_type        = "STANDARD"
    type              = "efs-ha"
    availability_zone = "cn-north-4a"
    flavor            = "sfs.turbo.20MBps"
  }
]

# Peer organizations configuration
peer_orgs = [
  {
    org_name = "organization"
    count    = 2
  }
]

# Channels configuration
channels = [
  {
    name      = "channel"
    org_names = ["organization"]
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="instance_name=basic-bcs-instance" -var="edition=4"`
2. Environment variables: `export TF_VAR_instance_name=basic-bcs-instance`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the BCS instance
4. Run `terraform show` to view the created BCS instance

## Reference Information

- [Huawei Cloud BCS Product Documentation](https://support.huaweicloud.com/bcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For BCS Basic Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/bcs/basic)
