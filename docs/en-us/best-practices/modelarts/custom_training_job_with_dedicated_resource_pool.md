# Deploy Custom Training Job with Dedicated Resource Pool

## Application Scenario

ModelArts is a model training and inference platform provided by Huawei Cloud for AI developers, supporting end-to-end AI development from data processing, algorithm development, model training to model deployment. Dedicated resource pools provide exclusive compute resources for ModelArts, suitable for enterprise AI training scenarios with high requirements for compute isolation, network configuration, and storage performance.

This best practice is suitable for scenarios where you need to create a dedicated resource pool on ModelArts and submit a custom training job, covering the complete deployment workflow of VPC network, SFS Turbo high-performance file storage, ModelArts network, workspace, dedicated resource pool, and training job. This best practice will introduce how to use Terraform to automatically deploy the above resources for end-to-end AI training infrastructure orchestration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zone Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ModelArts Resource Pool Flavor Query Data Source (data.huaweicloud_modelarts_resource_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelarts_resource_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule Resource (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [SFS Turbo File System Resource (huaweicloud_sfs_turbo)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sfs_turbo)
- [ModelArts Network Resource (huaweicloud_modelarts_network)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_network)
- [ModelArts Workspace Resource (huaweicloud_modelarts_workspace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_workspace)
- [ModelArts Dedicated Resource Pool Resource (huaweicloud_modelarts_resource_pool)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_resource_pool)
- [SMN Topic Resource (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [ModelArts Training Job Resource (huaweicloud_modelarts_training_job)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_training_job)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── huaweicloud_sfs_turbo
    └── huaweicloud_modelarts_resource_pool

data.huaweicloud_modelarts_resource_flavors
    └── huaweicloud_modelarts_resource_pool

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_sfs_turbo

huaweicloud_vpc_subnet
    └── huaweicloud_sfs_turbo

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule.test
    ├── huaweicloud_networking_secgroup_rule.udp_ingress_access
    └── huaweicloud_sfs_turbo

huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_sfs_turbo

huaweicloud_networking_secgroup_rule.udp_ingress_access
    └── huaweicloud_sfs_turbo

huaweicloud_sfs_turbo
    └── huaweicloud_modelarts_network

huaweicloud_modelarts_network
    └── huaweicloud_modelarts_resource_pool

huaweicloud_modelarts_workspace
    ├── huaweicloud_modelarts_resource_pool
    └── huaweicloud_modelarts_training_job

huaweicloud_modelarts_resource_pool
    └── huaweicloud_modelarts_training_job

huaweicloud_smn_topic
    └── huaweicloud_modelarts_training_job
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Query Availability Zone Information

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query availability zone information, which is used to create SFS Turbo file systems and dedicated resource pools:

```hcl
# Get all available availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create SFS Turbo file systems and dedicated resource pools
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- This data source requires no additional parameters and automatically queries all availability zones in the current region

### 3. Create VPC Network

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  type        = string
  description = "The VPC name"
}

variable "vpc_cidr" {
  type        = string
  default     = "192.168.0.0/16"
  description = "The CIDR block of the VPC"
}

variable "enterprise_project_id" {
  type        = string
  default     = null
  description = "The enterprise project ID"
}

# Create a VPC resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable vpc_cidr, default value is "192.168.0.0/16"
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is null

### 4. Create VPC Subnet

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  type        = string
  description = "The subnet name"
}

variable "subnet_cidr" {
  type        = string
  default     = ""
  nullable    = false
  description = "The CIDR block of the subnet"
}

variable "subnet_gateway_ip" {
  type        = string
  default     = ""
  nullable    = false
  description = "The gateway IP of the subnet"
}

# Create a VPC subnet resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **vpc_id**: VPC ID that the subnet belongs to, referencing the ID of the previously created VPC resource
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block, uses subnet_cidr when it is not empty, otherwise automatically calculated based on VPC CIDR
- **gateway_ip**: Subnet gateway IP, uses subnet_gateway_ip when it is not empty, otherwise automatically calculated based on subnet CIDR

### 5. Create Security Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  type        = string
  description = "The name of the security group"
}

# Create a security group resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: Security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to true to delete default security group rules

### 6. Create Security Group Ingress TCP Rule

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group ingress TCP rule for SFS Turbo access:

```hcl
# Create a security group ingress TCP rule under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used for SFS Turbo access
# Make sure open the full ingress access for 111, 2048, 2049, 2051, 2052 and 20048 ports and about TCP and UDP protocols.
resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  ports             = "111,2048,2049,2051,2052,20048"
}
```

**Parameter Description**:
- **security_group_id**: Security group ID, referencing the ID of the previously created security group resource
- **direction**: Rule direction, set to "ingress" for inbound traffic
- **ethertype**: Ethernet type, set to "IPv4"
- **protocol**: Protocol type, set to "tcp"
- **ports**: Open ports, set to "111,2048,2049,2051,2052,20048"

> SFS Turbo requires inbound TCP access on ports 111, 2048, 2049, 2051, 2052, and 20048. Follow the principle of least privilege and adjust ports and source address ranges based on business requirements during actual deployment.

### 7. Create Security Group Ingress UDP Rule

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group ingress UDP rule for SFS Turbo access:

```hcl
# Create a security group ingress UDP rule under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used for SFS Turbo access
resource "huaweicloud_networking_secgroup_rule" "udp_ingress_access" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "udp"
  ports             = "111,2048,2049,2051,2052,20048"
}
```

**Parameter Description**:
- **security_group_id**: Security group ID, referencing the ID of the previously created security group resource
- **direction**: Rule direction, set to "ingress" for inbound traffic
- **ethertype**: Ethernet type, set to "IPv4"
- **protocol**: Protocol type, set to "udp"
- **ports**: Open ports, set to "111,2048,2049,2051,2052,20048"

> SFS Turbo requires inbound UDP access on ports 111, 2048, 2049, 2051, 2052, and 20048. Follow the principle of least privilege and adjust ports and source address ranges based on business requirements during actual deployment.

### 8. Create SFS Turbo File System

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an SFS Turbo file system resource:

```hcl
variable "turbo_name" {
  type        = string
  description = "The name of the SFS Turbo"
}

variable "turbo_size" {
  type        = number
  default     = 1228
  description = "The size of the SFS Turbo"
}

variable "turbo_share_proto" {
  type        = string
  default     = "NFS"
  description = "The share protocol of the SFS Turbo"
}

variable "turbo_share_type" {
  type        = string
  default     = "HPC"
  description = "The share type of the SFS Turbo"
}

variable "turbo_hpc_bandwidth" {
  type        = string
  default     = "40M"
  description = "The HPC bandwidth of the SFS Turbo"
}

# Create an SFS Turbo file system resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_sfs_turbo" "test" {
  name                  = var.turbo_name
  size                  = var.turbo_size
  share_proto           = var.turbo_share_proto
  share_type            = var.turbo_share_type
  hpc_bandwidth         = var.turbo_hpc_bandwidth
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zone     = try(data.huaweicloud_availability_zones.test.names[0], null)
  enterprise_project_id = var.enterprise_project_id

  depends_on = [
    huaweicloud_networking_secgroup_rule.test,
    huaweicloud_networking_secgroup_rule.udp_ingress_access,
  ]

  lifecycle {
    ignore_changes = [
      availability_zone,
    ]
  }
}
```

**Parameter Description**:
- **name**: SFS Turbo name, assigned by referencing the input variable turbo_name
- **size**: SFS Turbo capacity, assigned by referencing the input variable turbo_size, default value is 1228
- **share_proto**: Share protocol, assigned by referencing the input variable turbo_share_proto, default value is "NFS"
- **share_type**: Share type, assigned by referencing the input variable turbo_share_type, default value is "HPC"
- **hpc_bandwidth**: HPC bandwidth, assigned by referencing the input variable turbo_hpc_bandwidth, default value is "40M"
- **vpc_id**: VPC ID, referencing the ID of the previously created VPC resource
- **subnet_id**: Subnet ID, referencing the ID of the previously created VPC subnet resource
- **security_group_id**: Security group ID, referencing the ID of the previously created security group resource
- **availability_zone**: Availability zone, assigned based on the return result of the availability zone query data source (data.huaweicloud_availability_zones)
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **depends_on**: Explicitly depends on security group rule resources to ensure security group rules are created before SFS Turbo
- **lifecycle.ignore_changes**: Lifecycle management, ignores changes to availability_zone

### 9. Create ModelArts Network

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a ModelArts network resource:

```hcl
variable "network_name" {
  type        = string
  description = "The name of the network"
}

variable "network_cidr" {
  type        = string
  default     = "10.168.0.0/16"
  description = "The CIDR block of the network"
}

# Create a ModelArts network resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_modelarts_network" "test" {
  name = var.network_name
  cidr = var.network_cidr # The recommended connecting CIDR about SFS Turbo.

  sfs_turbos {
    name = huaweicloud_sfs_turbo.test.name
    id   = huaweicloud_sfs_turbo.test.id
  }
}
```

**Parameter Description**:
- **name**: ModelArts network name, assigned by referencing the input variable network_name
- **cidr**: ModelArts network CIDR block, assigned by referencing the input variable network_cidr, default value is "10.168.0.0/16", recommended CIDR for connecting with SFS Turbo
- **sfs_turbos.name**: Associated SFS Turbo name, referencing the name of the previously created SFS Turbo resource
- **sfs_turbos.id**: Associated SFS Turbo ID, referencing the ID of the previously created SFS Turbo resource

### 10. Create ModelArts Workspace (Optional)

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a ModelArts workspace resource. Skip this step when an existing workspace is specified via workspace_id:

```hcl
variable "workspace_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the workspace to create. Cannot be configured together with workspace_id."

  validation {
    condition     = var.workspace_name == "" || var.workspace_id == ""
    error_message = "workspace_name and workspace_id cannot be configured at the same time."
  }
}

variable "workspace_id" {
  type        = string
  default     = ""
  description = "The existing workspace ID. Cannot be configured together with workspace_name."
}

# Create a ModelArts workspace resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_modelarts_workspace" "test" {
  count = var.workspace_name != "" ? 1 : 0

  name = var.workspace_name
}
```

**Parameter Description**:
- **count**: Number of resources to create, creates a workspace only when workspace_name is not empty
- **name**: Workspace name, assigned by referencing the input variable workspace_name

> workspace_name and workspace_id cannot be configured at the same time. To use an existing workspace, configure workspace_id and leave workspace_name empty.

### 11. Query ModelArts Dedicated Resource Pool Flavors

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query ModelArts dedicated resource pool flavor information. Skip this step when a flavor ID is specified via resource_pool_flavor_id:

```hcl
variable "resource_pool_flavor_id" {
  type        = string
  default     = ""
  nullable    = false
  description = "The flavor ID of the resource pool"
}

# Get ModelArts dedicated resource pool flavor information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create dedicated resource pools
data "huaweicloud_modelarts_resource_flavors" "test" {
  count = var.resource_pool_flavor_id != "" ? 0 : 1

  type = "Dedicate"
}
```

**Parameter Description**:
- **count**: Number of data sources to create, executes flavor query only when resource_pool_flavor_id is empty
- **type**: Resource pool type, set to "Dedicate" for dedicated resource pools

### 12. Create ModelArts Dedicated Resource Pool

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a ModelArts dedicated resource pool resource:

```hcl
variable "resource_pool_name" {
  type        = string
  description = "The name of the resource pool"
}

variable "resource_pool_scope" {
  type        = list(string)
  default     = ["Notebook", "Train", "Infer"]
  description = "The scope of the resource pool"
}

locals {
  available_resource_flavors = [
    for o in try(data.huaweicloud_modelarts_resource_flavors.test[0].flavors, []) : o if lookup(o.az_status, try(data.huaweicloud_availability_zones.test.names[0], null), "soldout") == "normal"
  ]
}

# Create a ModelArts dedicated resource pool resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_modelarts_resource_pool" "test" {
  name         = var.resource_pool_name
  scope        = var.resource_pool_scope
  network_id   = huaweicloud_modelarts_network.test.id
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(huaweicloud_modelarts_workspace.test[0].id, null)

  resources {
    flavor_id = var.resource_pool_flavor_id != "" ? var.resource_pool_flavor_id : try(local.available_resource_flavors[0].id, null)
    count     = 1
  }

  # If you want to change the `flavor` or other fields, you need to remove the corresponding fields from "lifecycle.ignore_changes".
  lifecycle {
    ignore_changes = [
      resources,
    ]
  }
}
```

**Parameter Description**:
- **name**: Dedicated resource pool name, assigned by referencing the input variable resource_pool_name
- **scope**: Resource pool scope, assigned by referencing the input variable resource_pool_scope, default value is ["Notebook", "Train", "Infer"]
- **network_id**: ModelArts network ID, referencing the ID of the previously created ModelArts network resource
- **workspace_id**: Workspace ID, uses workspace_id when it is not empty, otherwise references the created workspace resource ID
- **resources.flavor_id**: Resource pool flavor ID, uses resource_pool_flavor_id when it is not empty, otherwise selects the first available flavor from the available flavor list
- **resources.count**: Resource count, set to 1
- **lifecycle.ignore_changes**: Lifecycle management, ignores changes to resources. To modify flavor or other fields, remove the corresponding fields from ignore_changes

### 13. Create SMN Topic (Optional)

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an SMN topic resource for training job notifications. Skip this step when an existing topic is specified via training_job_notification_topic_urn:

```hcl
variable "topic_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the SMN topic to create for notifications. Cannot be configured together with training_job_notification_topic_urn."

  validation {
    condition     = var.topic_name == "" || var.training_job_notification_topic_urn == ""
    error_message = "topic_name and training_job_notification_topic_urn cannot be configured at the same time."
  }
}

variable "training_job_notification_topic_urn" {
  type        = string
  default     = ""
  nullable    = false
  description = "The existing SMN topic URN for training job notifications. Cannot be configured together with topic_name."
}

# Create an SMN topic resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_smn_topic" "test" {
  count = var.topic_name != "" ? 1 : 0

  name = var.topic_name
}
```

**Parameter Description**:
- **count**: Number of resources to create, creates an SMN topic only when topic_name is not empty
- **name**: SMN topic name, assigned by referencing the input variable topic_name

> topic_name and training_job_notification_topic_urn cannot be configured at the same time.

### 14. Create ModelArts Custom Training Job

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a ModelArts custom training job resource:

```hcl
variable "training_job_name" {
  type        = string
  description = "The name of the training job"
}

variable "training_job_annotations" {
  type        = map(string)
  default     = {}
  description = "The annotations of the training job"
}

variable "training_job_description" {
  type        = string
  default     = null
  description = "The description of the training job"
}

variable "training_job_code_dir" {
  type        = string
  description = "The OBS code directory of the training job"
}

variable "training_job_command" {
  type        = string
  description = "The container startup command for the training job"
}

variable "training_job_engine" {
  type = object({
    image_url = optional(string)
    id        = optional(string)
    version   = optional(string)
    name      = optional(string)
  })

  description = "The engine configuration of the training job"
}

variable "training_job_inputs" {
  type = list(object({
    local_dir = string

    dataset = object({
      id           = string
      name         = optional(string)
      version_id   = optional(string)
      service_type = optional(string)
    })
  }))

  default     = []
  nullable    = false
  description = "The inputs of the training job"
}

variable "training_job_environments" {
  type        = map(string)
  default     = {}
  nullable    = false
  description = "The environment variables of the training job"
}

variable "resource_node_count" {
  type        = number
  default     = 1
  description = "The number of resource replicas used by the training job"
}

variable "training_job_volumes" {
  type = list(object({
    nfs = optional(object({
      nfs_server_path = optional(string)
      local_path      = optional(string)
      read_only       = optional(bool)
    }))

    pfs = optional(object({
      pfs_path   = optional(string)
      local_path = optional(string)
    }))

    obs = optional(object({
      obs_path   = optional(string)
      local_path = optional(string)
    }))
  }))

  default     = []
  nullable    = false
  description = "The volume mount configuration of the training job"
}

variable "training_job_log_export_path_obs_url" {
  type        = string
  default     = ""
  nullable    = false
  description = "The log export path of the training job"
}

variable "training_job_log_export_config_version" {
  type        = string
  default     = ""
  nullable    = false
  description = "The log export config version of the training job"
}

variable "training_job_auto_stop_duration" {
  type        = number
  default     = 0
  description = "The auto stop duration of the training job"
}

variable "training_job_notification_events" {
  type        = list(string)
  default     = []
  description = "The notification events of the training job"
}

variable "custom_metrics" {
  type = object({
    exec = optional(object({
      command = list(string)
    }))

    http_get = optional(object({
      path = string
      port = number
    }))
  })

  default     = null
  description = "The custom metrics of the training job"
}

variable "training_job_asset_model" {
  type = object({
    name    = string
    version = string
    type    = string
    code    = optional(string)
    desc    = optional(string)
    series  = optional(string)
  })

  default     = null
  description = "The asset model of the training job"
}

variable "training_job_output_model" {
  type = object({
    obs_path   = string
    local_path = optional(string)
  })

  default     = null
  description = "The output model of the training job"
}

variable "training_job_tags" {
  type        = map(string)
  default     = {}
  description = "The tags of the training job"
}

# Create a ModelArts custom training job resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_modelarts_training_job" "test" {
  kind = "job"

  metadata {
    name         = var.training_job_name
    workspace_id = var.workspace_id != "" ? var.workspace_id : try(huaweicloud_modelarts_workspace.test[0].id, null)
    annotations  = var.training_job_annotations
    description  = var.training_job_description
  }

  algorithm {
    code_dir = var.training_job_code_dir
    command  = var.training_job_command

    engine {
      image_url      = var.training_job_engine.image_url
      engine_id      = var.training_job_engine.id
      engine_version = var.training_job_engine.version
      engine_name    = var.training_job_engine.name
    }


    dynamic "inputs" {
      for_each = var.training_job_inputs

      content {
        local_dir = inputs.value.local_dir

        remote {
          dataset {
            id           = inputs.value.dataset.id
            name         = inputs.value.dataset.name
            version_id   = inputs.value.dataset.version_id
            service_type = inputs.value.dataset.service_type
          }
        }
      }
    }

    environments = var.training_job_environments
  }

  spec {
    resource {
      pool_id    = huaweicloud_modelarts_resource_pool.test.id
      node_count = var.resource_node_count
    }

    dynamic "volumes" {
      for_each = var.training_job_volumes

      content {
        dynamic "nfs" {
          for_each = volumes.value.nfs != null ? [volumes.value.nfs] : []

          content {
            nfs_server_path = nfs.value.nfs_server_path
            local_path      = nfs.value.local_path
            read_only       = nfs.value.read_only
          }
        }

        dynamic "pfs" {
          for_each = volumes.value.pfs != null ? [volumes.value.pfs] : []

          content {
            pfs_path   = pfs.value.pfs_path
            local_path = pfs.value.local_path
          }
        }

        dynamic "obs" {
          for_each = volumes.value.obs != null ? [volumes.value.obs] : []

          content {
            obs_path   = obs.value.obs_path
            local_path = obs.value.local_path
          }
        }
      }
    }

    dynamic "log_export_path" {
      for_each = var.training_job_log_export_path_obs_url != "" ? [1] : []

      content {
        obs_url = var.training_job_log_export_path_obs_url
      }
    }

    dynamic "log_export_config" {
      for_each = var.training_job_log_export_config_version != "" ? [1] : []

      content {
        version = var.training_job_log_export_config_version
      }
    }

    dynamic "auto_stop" {
      for_each = var.training_job_auto_stop_duration != 0 ? [1] : []

      content {
        time_unit = "HOURS"
        duration  = var.training_job_auto_stop_duration
      }
    }

    dynamic "notification" {
      for_each = var.training_job_notification_topic_urn != "" || var.topic_name != "" ? [1] : []

      content {
        topic_urn = var.training_job_notification_topic_urn != "" ? var.training_job_notification_topic_urn : var.topic_name != "" ? huaweicloud_smn_topic.test[0].id : ""
        events    = var.training_job_notification_events
      }
    }

    dynamic "custom_metrics" {
      for_each = var.custom_metrics != null ? [var.custom_metrics] : []

      content {
        dynamic "exec" {
          for_each = custom_metrics.value.exec != null ? [custom_metrics.value.exec] : []

          content {
            command = exec.value.command
          }
        }

        dynamic "http_get" {
          for_each = custom_metrics.value.http_get != null ? [custom_metrics.value.http_get] : []
          content {
            path = http_get.value.path
            port = http_get.value.port
          }
        }
      }
    }

    dynamic "asset_model" {
      for_each = var.training_job_asset_model != null ? [var.training_job_asset_model] : []

      content {
        name    = asset_model.value.name
        version = asset_model.value.version
        type    = asset_model.value.type
        code    = asset_model.value.code
        desc    = asset_model.value.desc
        series  = asset_model.value.series
      }
    }

    dynamic "output_model" {
      for_each = var.training_job_output_model != null ? [var.training_job_output_model] : []

      content {
        obs {
          obs_path   = output_model.value.obs_path
          local_path = output_model.value.local_path
        }
      }
    }
  }

  tags = var.training_job_tags
}
```

**Parameter Description**:
- **kind**: Training job type, set to "job" for custom training jobs
- **metadata.name**: Training job name, assigned by referencing the input variable training_job_name
- **metadata.workspace_id**: Workspace ID, uses workspace_id when it is not empty, otherwise references the created workspace resource ID
- **metadata.annotations**: Training job annotations, assigned by referencing the input variable training_job_annotations, default value is an empty map
- **metadata.description**: Training job description, assigned by referencing the input variable training_job_description
- **algorithm.code_dir**: OBS directory of training code, assigned by referencing the input variable training_job_code_dir
- **algorithm.command**: Container startup command, assigned by referencing the input variable training_job_command
- **algorithm.engine**: Training engine configuration, assigned by referencing the input variable training_job_engine, supporting fields such as image_url, id, version, and name
- **algorithm.inputs**: Training input configuration, assigned by referencing the input variable training_job_inputs, supporting dataset mounting
- **algorithm.environments**: Environment variables, assigned by referencing the input variable training_job_environments, default value is an empty map
- **spec.resource.pool_id**: Dedicated resource pool ID, referencing the ID of the previously created dedicated resource pool resource
- **spec.resource.node_count**: Number of resource replicas, assigned by referencing the input variable resource_node_count, default value is 1
- **spec.volumes**: Volume mount configuration, assigned by referencing the input variable training_job_volumes, supporting NFS, PFS, and OBS mounts
- **spec.log_export_path**: Log export path, configured when training_job_log_export_path_obs_url is not empty
- **spec.log_export_config**: Log export configuration, configured when training_job_log_export_config_version is not empty
- **spec.auto_stop**: Auto stop configuration, configured when training_job_auto_stop_duration is not 0, time unit is HOURS
- **spec.notification**: Notification configuration, configured when training_job_notification_topic_urn or topic_name is not empty
- **spec.custom_metrics**: Custom metrics configuration, assigned by referencing the input variable custom_metrics
- **spec.asset_model**: Asset model configuration, assigned by referencing the input variable training_job_asset_model
- **spec.output_model**: Output model configuration, assigned by referencing the input variable training_job_output_model
- **tags**: Training job tags, assigned by referencing the input variable training_job_tags, default value is an empty map

### 15. Preset Input Parameters for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration values. These input parameters need to be entered manually during subsequent deployment.
Terraform also provides a method to preset these configurations through a `terraform.tfvars` file to avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC network configuration
vpc_name              = "tf_test_vpc"
subnet_name           = "tf_test_subnet"
security_group_name   = "tf_test_security_group"

# SFS Turbo configuration
turbo_name = "tf_test_sfs_turbo"

# ModelArts network and resource pool configuration
network_name            = "tf-test-network"
resource_pool_name      = "tf-test-resource-pool"
resource_pool_flavor_id = "your_resource_pool_flavor_id"

# Training job configuration
training_job_name     = "tf_test_training_job"
training_job_code_dir = "your_training_code_dir"
training_job_command  = "your_training_command"

training_job_engine = {
  image_url = "your_swr_image_url"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the contents when executing terraform commands; other names require adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read variable values from this file

In addition to using a `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="training_job_name=my-job"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform uses variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 16. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the dedicated resource pool and custom training job
4. Run `terraform show` to view details of the created dedicated resource pool and custom training job

## Reference Information

- [Huawei Cloud ModelArts Product Documentation](https://support.huaweicloud.com/modelarts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ModelArts Custom Training Job with Dedicated Resource Pool](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/modelarts/custom-training-job-with-dedicated-resource-pool)
