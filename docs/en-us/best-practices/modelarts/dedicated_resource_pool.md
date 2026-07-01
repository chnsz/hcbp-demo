# Deploy Dedicated Resource Pool

## Application Scenario

The ModelArts dedicated resource pool provides exclusive compute resources for AI training, inference, and Notebook development. It supports deep integration with VPC, SFS Turbo high-performance file storage, and ModelArts networks to meet enterprise AI requirements for network isolation, storage performance, and compute capacity.

This best practice applies to creating a ModelArts dedicated resource pool. It supports two deployment modes: a full deployment that creates VPC network, SFS Turbo, ModelArts network, and dedicated resource pool together; or using an existing ModelArts network and SFS Turbo connection information to create only the dedicated resource pool. This best practice introduces how to use Terraform to automatically deploy the above resources and manage dedicated resource pools with Infrastructure as Code.

## Related Resources and Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ModelArts Resource Flavors Data Source (data.huaweicloud_modelarts_resource_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelarts_resource_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule Resource (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [SFS Turbo File System Resource (huaweicloud_sfs_turbo)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sfs_turbo)
- [ModelArts Network Resource (huaweicloud_modelarts_network)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_network)
- [ModelArts Workspace Resource (huaweicloud_modelarts_workspace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_workspace)
- [ModelArts Dedicated Resource Pool Resource (huaweicloud_modelarts_resource_pool)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_resource_pool)

### Resource/Data Source Dependency Relationships

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
    └── huaweicloud_modelarts_resource_pool
```

## Operation Steps

### 1. Script Preparation

Prepare TF files (such as main.tf) in the specified workspace for writing the current best practice scripts. Ensure that they (or other TF files in the same directory) contain the provider version declaration and Huawei Cloud authentication information required for resource deployment.
For configuration details, refer to [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zone Information

Add the following script to the TF file (such as main.tf) to instruct Terraform to query availability zone information. The query results are used to create SFS Turbo file systems and filter dedicated resource pool flavors:

```hcl
# Query all availability zones in the specified region (inherits the region from the current provider block when the region parameter is omitted), used for creating SFS Turbo file systems and dedicated resource pools
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- This data source requires no additional parameters and automatically queries all availability zones in the current region

### 3. Create VPC Network (Optional)

Add the following script to the TF file (such as main.tf) to instruct Terraform to create a VPC resource. Created when a new SFS Turbo is needed (i.e., when turbo_name is specified):

```hcl
variable "turbo_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the SFS Turbo to create"

  validation {
    condition     = var.network_name != "" || var.turbo_name == ""
    error_message = "turbo_name can only be specified when network_name is specified."
  }
}

variable "vpc_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The VPC name. Can only be specified when turbo_name is specified."

  validation {
    condition     = (var.turbo_name != "") == (var.vpc_name != "")
    error_message = "vpc_name can only be specified when turbo_name is specified."
  }
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

# Create a VPC resource in the specified region (inherits the region from the current provider block when the region parameter is omitted)
resource "huaweicloud_vpc" "test" {
  count = var.turbo_name != "" ? 1 : 0

  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **count**: Number of resources to create. VPC is created only when turbo_name is not empty
- **name**: VPC name, assigned by referencing the input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable vpc_cidr, default value is "192.168.0.0/16"
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is null

> turbo_name can only be specified when network_name is also specified. vpc_name must be specified together with turbo_name or left empty together.

### 4. Create VPC Subnet (Optional)

Add the following script to the TF file (such as main.tf) to instruct Terraform to create a VPC subnet resource. Created when a new SFS Turbo is needed:

```hcl
variable "subnet_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The subnet name. Can only be specified when turbo_name is specified."

  validation {
    condition     = (var.turbo_name != "") == (var.subnet_name != "")
    error_message = "subnet_name can only be specified when turbo_name is specified."
  }
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

# Create a VPC subnet resource in the specified region (inherits the region from the current provider block when the region parameter is omitted)
resource "huaweicloud_vpc_subnet" "test" {
  count = var.turbo_name != "" ? 1 : 0

  vpc_id     = huaweicloud_vpc.test[0].id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test[0].cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test[0].cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **count**: Number of resources to create. Subnet is created only when turbo_name is not empty
- **vpc_id**: VPC ID to which the subnet belongs, references the ID of the previously created VPC resource
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block. Uses subnet_cidr when not empty, otherwise automatically calculated based on VPC CIDR
- **gateway_ip**: Subnet gateway IP. Uses subnet_gateway_ip when not empty, otherwise automatically calculated based on subnet CIDR

### 5. Create Security Group (Optional)

Add the following script to the TF file (such as main.tf) to instruct Terraform to create a security group resource. Created when a new SFS Turbo is needed:

```hcl
variable "security_group_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the security group"

  validation {
    condition     = (var.turbo_name != "") == (var.security_group_name != "")
    error_message = "security_group_name can only be specified when turbo_name is specified."
  }
}

# Create a security group resource in the specified region (inherits the region from the current provider block when the region parameter is omitted)
resource "huaweicloud_networking_secgroup" "test" {
  count = var.turbo_name != "" ? 1 : 0

  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **count**: Number of resources to create. Security group is created only when turbo_name is not empty
- **name**: Security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules. Set to true to delete default security group rules

### 6. Create Security Group Ingress TCP Rule (Optional)

Add the following script to the TF file (such as main.tf) to instruct Terraform to create a security group ingress TCP rule for SFS Turbo access. Created when a new SFS Turbo is needed:

```hcl
# Create a security group ingress TCP rule in the specified region (inherits the region from the current provider block when the region parameter is omitted) for SFS Turbo access
# Make sure open the full ingress access for 111, 2048, 2049, 2051, 2052 and 20048 ports and about TCP and UDP protocols.
resource "huaweicloud_networking_secgroup_rule" "test" {
  count = var.turbo_name != "" ? 1 : 0

  security_group_id = huaweicloud_networking_secgroup.test[0].id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  ports             = "111,2048,2049,2051,2052,20048"
}
```

**Parameter Description**:
- **count**: Number of resources to create. Rule is created only when turbo_name is not empty
- **security_group_id**: Security group ID, references the ID of the previously created security group resource
- **direction**: Rule direction, set to "ingress" for inbound traffic
- **ethertype**: Ethernet type, set to "IPv4"
- **protocol**: Protocol type, set to "tcp"
- **ports**: Open ports, set to "111,2048,2049,2051,2052,20048"

> SFS Turbo requires TCP ingress access on ports 111, 2048, 2049, 2051, 2052, and 20048. Follow the principle of least privilege and adjust ports and source address ranges according to business needs during actual deployment.

### 7. Create Security Group Ingress UDP Rule (Optional)

Add the following script to the TF file (such as main.tf) to instruct Terraform to create a security group ingress UDP rule for SFS Turbo access. Created when a new SFS Turbo is needed:

```hcl
# Create a security group ingress UDP rule in the specified region (inherits the region from the current provider block when the region parameter is omitted) for SFS Turbo access
resource "huaweicloud_networking_secgroup_rule" "udp_ingress_access" {
  count = var.turbo_name != "" ? 1 : 0

  security_group_id = huaweicloud_networking_secgroup.test[0].id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "udp"
  ports             = "111,2048,2049,2051,2052,20048"
}
```

**Parameter Description**:
- **count**: Number of resources to create. Rule is created only when turbo_name is not empty
- **security_group_id**: Security group ID, references the ID of the previously created security group resource
- **direction**: Rule direction, set to "ingress" for inbound traffic
- **ethertype**: Ethernet type, set to "IPv4"
- **protocol**: Protocol type, set to "udp"
- **ports**: Open ports, set to "111,2048,2049,2051,2052,20048"

> SFS Turbo requires UDP ingress access on ports 111, 2048, 2049, 2051, 2052, and 20048. Follow the principle of least privilege and adjust ports and source address ranges according to business needs during actual deployment.

### 8. Create SFS Turbo File System (Optional)

Add the following script to the TF file (such as main.tf) to instruct Terraform to create an SFS Turbo file system resource. Created when turbo_name is specified:

```hcl
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

# Create an SFS Turbo file system resource in the specified region (inherits the region from the current provider block when the region parameter is omitted)
resource "huaweicloud_sfs_turbo" "test" {
  count = var.turbo_name != "" ? 1 : 0

  name                  = var.turbo_name
  size                  = var.turbo_size
  share_proto           = var.turbo_share_proto
  share_type            = var.turbo_share_type
  hpc_bandwidth         = var.turbo_hpc_bandwidth
  vpc_id                = huaweicloud_vpc.test[0].id
  subnet_id             = huaweicloud_vpc_subnet.test[0].id
  security_group_id     = huaweicloud_networking_secgroup.test[0].id
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
- **count**: Number of resources to create. SFS Turbo is created only when turbo_name is not empty
- **name**: SFS Turbo name, assigned by referencing the input variable turbo_name
- **size**: SFS Turbo capacity, assigned by referencing the input variable turbo_size, default value is 1228
- **share_proto**: Share protocol, assigned by referencing the input variable turbo_share_proto, default value is "NFS"
- **share_type**: Share type, assigned by referencing the input variable turbo_share_type, default value is "HPC"
- **hpc_bandwidth**: HPC bandwidth, assigned by referencing the input variable turbo_hpc_bandwidth, default value is "40M"
- **vpc_id**: VPC ID, references the ID of the previously created VPC resource
- **subnet_id**: Subnet ID, references the ID of the previously created VPC subnet resource
- **security_group_id**: Security group ID, references the ID of the previously created security group resource
- **availability_zone**: Availability zone, assigned based on the return result of the availability zones data source (data.huaweicloud_availability_zones)
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **depends_on**: Explicit dependency on security group rule resources to ensure security group rules are created before SFS Turbo
- **lifecycle.ignore_changes**: Lifecycle management, ignores changes to availability_zone

### 9. Create ModelArts Network (Optional)

Add the following script to the TF file (such as main.tf) to instruct Terraform to create a ModelArts network resource. Created when network_name is specified. If using an existing network, configure network_id and leave network_name empty:

```hcl
variable "network_sfs_turbos" {
  type = list(object({
    id   = string
    name = string
  }))

  default     = []
  nullable    = false
  description = "The SFS Turbo connections for the ModelArts network"

  validation {
    condition     = length(var.network_sfs_turbos) == 0 || (var.network_name != "" && var.turbo_name == "")
    error_message = "network_sfs_turbos can only be specified when network_name is specified and turbo_name is not specified."
  }
}

variable "network_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the ModelArts network to create"

  validation {
    condition     = (var.network_id != "") != (var.network_name != "")
    error_message = "Exactly one of network_id and network_name must be specified."
  }
}

variable "network_cidr" {
  type        = string
  default     = "10.168.0.0/16"
  description = "The CIDR block of the ModelArts network"
}

variable "network_id" {
  type        = string
  default     = ""
  nullable    = false
  description = "The existing ModelArts network ID"
}

locals {
  network_sfs_turbos = var.turbo_name != "" ? [
    {
      id   = huaweicloud_sfs_turbo.test[0].id
      name = huaweicloud_sfs_turbo.test[0].name
    }
  ] : var.network_sfs_turbos
}

# Create a ModelArts network resource in the specified region (inherits the region from the current provider block when the region parameter is omitted)
resource "huaweicloud_modelarts_network" "test" {
  count = var.network_name != "" ? 1 : 0

  name = var.network_name
  cidr = var.network_cidr

  dynamic "sfs_turbos" {
    for_each = local.network_sfs_turbos

    content {
      id   = sfs_turbos.value.id
      name = sfs_turbos.value.name
    }
  }
}
```

**Parameter Description**:
- **count**: Number of resources to create. ModelArts network is created only when network_name is not empty
- **name**: ModelArts network name, assigned by referencing the input variable network_name
- **cidr**: ModelArts network CIDR block, assigned by referencing the input variable network_cidr, default value is "10.168.0.0/16"
- **sfs_turbos.id**: Associated SFS Turbo ID. References the created SFS Turbo resource ID when creating a new SFS Turbo, otherwise specified via the network_sfs_turbos variable
- **sfs_turbos.name**: Associated SFS Turbo name. References the created SFS Turbo resource name when creating a new SFS Turbo, otherwise specified via the network_sfs_turbos variable

> Exactly one of network_id and network_name must be specified. When using an existing SFS Turbo, specify connection information via network_sfs_turbos. turbo_name cannot be specified at the same time.

### 10. Create ModelArts Workspace (Optional)

Add the following script to the TF file (such as main.tf) to instruct Terraform to create a ModelArts workspace resource. Skip this step when an existing workspace is specified via workspace_id:

```hcl
variable "workspace_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the workspace to create"

  validation {
    condition     = var.workspace_name == "" || var.workspace_id == ""
    error_message = "workspace_name and workspace_id cannot be configured at the same time."
  }
}

variable "workspace_id" {
  type        = string
  default     = ""
  description = "The existing workspace ID"
}

# Create a ModelArts workspace resource in the specified region (inherits the region from the current provider block when the region parameter is omitted)
resource "huaweicloud_modelarts_workspace" "test" {
  count = var.workspace_name != "" ? 1 : 0

  name = var.workspace_name
}
```

**Parameter Description**:
- **count**: Number of resources to create. Workspace is created only when workspace_name is not empty
- **name**: Workspace name, assigned by referencing the input variable workspace_name

> workspace_name and workspace_id cannot be configured at the same time. If using an existing workspace, configure workspace_id and leave workspace_name empty.

### 11. Query ModelArts Dedicated Resource Pool Flavors

Add the following script to the TF file (such as main.tf) to instruct Terraform to query ModelArts dedicated resource pool flavor information. Skip this step when flavor_id is specified for all resources in resource_pool_resources:

```hcl
variable "resource_pool_resources" {
  type = list(object({
    flavor_id     = optional(string, "")
    count         = number
    max_count     = optional(number)
    extend_params = optional(string)

    root_volume = optional(object({
      volume_type = string
      size        = string
    }))

    data_volumes = optional(list(object({
      volume_type   = string
      size          = string
      count         = optional(number)
      extend_params = optional(string)
    })), [])

    volume_group_configs = optional(list(object({
      volume_group     = string
      docker_thin_pool = optional(number)
      types            = optional(list(string))

      lvm_config = optional(object({
        lv_type = string
        path    = optional(string)
      }))
    })), [])

    os = optional(object({
      name       = optional(string)
      image_id   = optional(string)
      image_type = optional(string)
    }))

    driver = optional(object({
      version = string
    }))

    creating_step = optional(object({
      step = number
      type = string
    }))
  }))

  description = "The list of resource specifications in the resource pool"
}

locals {
  is_query_resource_flavor = length([for r in var.resource_pool_resources : r if r.flavor_id == ""]) > 0
}

# Query ModelArts dedicated resource pool flavor information in the specified region (inherits the region from the current provider block when the region parameter is omitted), used for creating dedicated resource pools
data "huaweicloud_modelarts_resource_flavors" "test" {
  count = local.is_query_resource_flavor ? 1 : 0

  type = "Dedicate"
}

locals {
  available_resource_flavors = [
    for o in try(data.huaweicloud_modelarts_resource_flavors.test[0].flavors, []) : o if lookup(o.az_status, try(data.huaweicloud_availability_zones.test.names[0], null), "soldout") == "normal"
  ]
}
```

**Parameter Description**:
- **count**: Number of data sources to create. Flavor query is executed only when flavor_id is empty for any resource specification in resource_pool_resources
- **type**: Resource pool type, set to "Dedicate" for dedicated resource pools
- **is_query_resource_flavor**: Local variable that determines whether resource pool flavor query is needed
- **available_resource_flavors**: Local variable that filters dedicated resource pool flavors with normal status in the current availability zone

### 12. Create ModelArts Dedicated Resource Pool

Add the following script to the TF file (such as main.tf) to instruct Terraform to create a ModelArts dedicated resource pool resource:

```hcl
variable "resource_pool_name" {
  type        = string
  description = "The name of the dedicated resource pool"
}

variable "resource_pool_description" {
  type        = string
  default     = null
  description = "The description of the dedicated resource pool"
}

variable "resource_pool_scope" {
  type        = list(string)
  default     = ["Train", "Infer", "Notebook"]
  description = "The scope of the dedicated resource pool"
}

variable "resource_pool_metadata_annotations" {
  type        = string
  default     = null
  description = "The annotations of the resource pool, in JSON format"
}

# Create a ModelArts dedicated resource pool resource in the specified region (inherits the region from the current provider block when the region parameter is omitted)
resource "huaweicloud_modelarts_resource_pool" "test" {
  name         = var.resource_pool_name
  description  = var.resource_pool_description
  scope        = var.resource_pool_scope
  network_id   = var.network_id != "" ? var.network_id : huaweicloud_modelarts_network.test[0].id
  workspace_id = var.workspace_id != "" ? var.workspace_id : try(huaweicloud_modelarts_workspace.test[0].id, null)

  dynamic "metadata" {
    for_each = var.resource_pool_metadata_annotations != null ? [1] : []

    content {
      annotations = var.resource_pool_metadata_annotations
    }
  }

  dynamic "resources" {
    for_each = var.resource_pool_resources

    content {
      flavor_id     = resources.value.flavor_id != "" ? resources.value.flavor_id : try(local.available_resource_flavors[0].id, null)
      count         = resources.value.count
      max_count     = resources.value.max_count
      extend_params = resources.value.extend_params

      dynamic "root_volume" {
        for_each = try(resources.value.root_volume, null) != null ? [resources.value.root_volume] : []

        content {
          volume_type = root_volume.value.volume_type
          size        = root_volume.value.size
        }
      }

      dynamic "data_volumes" {
        for_each = try(resources.value.data_volumes, [])

        content {
          volume_type   = data_volumes.value.volume_type
          size          = data_volumes.value.size
          extend_params = data_volumes.value.extend_params
          count         = try(data_volumes.value.count, null)
        }
      }

      dynamic "volume_group_configs" {
        for_each = try(resources.value.volume_group_configs, [])

        content {
          volume_group     = volume_group_configs.value.volume_group
          docker_thin_pool = try(volume_group_configs.value.docker_thin_pool, null)
          types            = try(volume_group_configs.value.types, null)

          dynamic "lvm_config" {
            for_each = volume_group_configs.value.lvm_config != null ? [volume_group_configs.value.lvm_config] : []

            content {
              lv_type = lvm_config.value.lv_type
              path    = try(lvm_config.value.path, null)
            }
          }
        }
      }

      dynamic "os" {
        for_each = resources.value.os != null ? [resources.value.os] : []

        content {
          name       = try(os.value.name, null)
          image_id   = try(os.value.image_id, null)
          image_type = try(os.value.image_type, null)
        }
      }

      dynamic "driver" {
        for_each = resources.value.driver != null ? [resources.value.driver] : []

        content {
          version = driver.value.version
        }
      }

      dynamic "creating_step" {
        for_each = resources.value.creating_step != null ? [resources.value.creating_step] : []

        content {
          step = creating_step.value.step
          type = creating_step.value.type
        }
      }
    }
  }
}
```

**Parameter Description**:
- **name**: Dedicated resource pool name, assigned by referencing the input variable resource_pool_name
- **description**: Dedicated resource pool description, assigned by referencing the input variable resource_pool_description, default value is null
- **scope**: Resource pool usage scope, assigned by referencing the input variable resource_pool_scope, default value is ["Train", "Infer", "Notebook"]
- **network_id**: ModelArts network ID. Uses its value when network_id is not empty, otherwise references the created ModelArts network resource ID
- **workspace_id**: Workspace ID. Uses its value when workspace_id is not empty, otherwise references the created workspace resource ID
- **metadata.annotations**: Resource pool metadata annotations, assigned by referencing the input variable resource_pool_metadata_annotations, in JSON format
- **resources.flavor_id**: Resource flavor ID. Uses its value when flavor_id is not empty, otherwise selects the first available flavor from the available flavor list
- **resources.count**: Resource count, assigned from the count in the corresponding item in resource_pool_resources
- **resources.max_count**: Maximum resource count, assigned from the max_count in the corresponding item in resource_pool_resources
- **resources.extend_params**: Extension parameters, assigned from the extend_params in the corresponding item in resource_pool_resources
- **resources.root_volume.volume_type**: System disk type, assigned from the root_volume.volume_type in the corresponding item in resource_pool_resources
- **resources.root_volume.size**: System disk size, assigned from the root_volume.size in the corresponding item in resource_pool_resources
- **resources.data_volumes.volume_type**: Data disk type, assigned from the data_volumes.volume_type in the corresponding item in resource_pool_resources
- **resources.data_volumes.size**: Data disk size, assigned from the data_volumes.size in the corresponding item in resource_pool_resources
- **resources.data_volumes.extend_params**: Data disk extension parameters, assigned from the data_volumes.extend_params in the corresponding item in resource_pool_resources
- **resources.data_volumes.count**: Data disk count, assigned from the data_volumes.count in the corresponding item in resource_pool_resources
- **resources.volume_group_configs.volume_group**: Volume group name, assigned from the volume_group_configs.volume_group in the corresponding item in resource_pool_resources
- **resources.volume_group_configs.docker_thin_pool**: Docker thin pool size, assigned from the volume_group_configs.docker_thin_pool in the corresponding item in resource_pool_resources
- **resources.volume_group_configs.types**: Volume group type list, assigned from the volume_group_configs.types in the corresponding item in resource_pool_resources
- **resources.volume_group_configs.lvm_config.lv_type**: LVM logical volume type, assigned from the volume_group_configs.lvm_config.lv_type in the corresponding item in resource_pool_resources
- **resources.volume_group_configs.lvm_config.path**: LVM logical volume path, assigned from the volume_group_configs.lvm_config.path in the corresponding item in resource_pool_resources
- **resources.os.name**: Operating system name, assigned from the os.name in the corresponding item in resource_pool_resources
- **resources.os.image_id**: Operating system image ID, assigned from the os.image_id in the corresponding item in resource_pool_resources
- **resources.os.image_type**: Operating system image type, assigned from the os.image_type in the corresponding item in resource_pool_resources
- **resources.driver.version**: Driver version, assigned from the driver.version in the corresponding item in resource_pool_resources
- **resources.creating_step.step**: Creation step number, assigned from the creating_step.step in the corresponding item in resource_pool_resources
- **resources.creating_step.type**: Creation step type, assigned from the creating_step.type in the corresponding item in resource_pool_resources

### 13. Preset Input Parameters for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration values. These input parameters need to be entered manually during subsequent deployment.
Meanwhile, Terraform provides a method to preset these configurations through the `terraform.tfvars` file to avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC network configuration
vpc_name            = "tf-test-vpc"
subnet_name         = "tf-test-subnet"
security_group_name = "tf-test-security-group"

# SFS Turbo configuration
turbo_name = "tf-test-sfs-turbo"

# ModelArts network and dedicated resource pool configuration
network_name              = "tf-test-network"
resource_pool_name        = "tf-test-resource-pool"
resource_pool_description = "This is a demo"

resource_pool_resources = [
  {
    count = 1
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the contents of this `tfvars` file when executing terraform commands; other names require adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read variable values from this file

In addition to using the `terraform.tfvars` file, variable values can also be set through the following methods:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="resource_pool_name=my-pool"`
2. Environment variables: `export TF_VAR_resource_pool_name=my-pool`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform uses variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 14. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the dedicated resource pool
4. Run `terraform show` to view the details of the created dedicated resource pool

## Reference Information

- [Huawei Cloud ModelArts Product Documentation](https://support.huaweicloud.com/modelarts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ModelArts Dedicated Resource Pool](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/modelarts/dedicated-resource-pool)
