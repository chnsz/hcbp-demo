# Deploy Notebook with Dedicated Resource Pool

## Application Scenario

ModelArts Notebook is an online AI development environment provided by Huawei Cloud, supporting Jupyter Notebook interactive development for algorithm engineers to perform data exploration, model development, and debugging. Dedicated resource pools provide exclusive compute resources for Notebooks, combined with SFS Turbo high-performance file storage to meet enterprise AI development requirements for network isolation, storage performance, and compute assurance.

This best practice is suitable for scenarios where you need to create a dedicated resource pool on ModelArts and deploy a Notebook instance, covering the complete deployment workflow of VPC network, SFS Turbo, ModelArts network, workspace, dedicated resource pool, Notebook flavor and image queries, key pair, and Notebook instance. This best practice will introduce how to use Terraform to automatically deploy the above resources for Infrastructure as Code management of Notebook development environments.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zone Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ModelArts Resource Pool Flavor Query Data Source (data.huaweicloud_modelarts_resource_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelarts_resource_flavors)
- [ModelArts Notebook Flavor Query Data Source (data.huaweicloud_modelarts_notebook_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelarts_notebook_flavors)
- [ModelArts Notebook Image Query Data Source (data.huaweicloud_modelarts_notebook_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelarts_notebook_images)
- [ModelArts V2 Resource Pool Query Data Source (data.huaweicloud_modelartsv2_resource_pools)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/modelartsv2_resource_pools)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule Resource (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [SFS Turbo File System Resource (huaweicloud_sfs_turbo)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sfs_turbo)
- [ModelArts Network Resource (huaweicloud_modelarts_network)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_network)
- [ModelArts Workspace Resource (huaweicloud_modelarts_workspace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_workspace)
- [ModelArts Dedicated Resource Pool Resource (huaweicloud_modelarts_resource_pool)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_resource_pool)
- [KPS Key Pair Resource (huaweicloud_kps_keypair)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [ModelArts Notebook Resource (huaweicloud_modelarts_notebook)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_notebook)
- [ModelArts Notebook Mount Storage Resource (huaweicloud_modelarts_notebook_mount_storage)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/modelarts_notebook_mount_storage)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── huaweicloud_sfs_turbo
    └── huaweicloud_modelarts_resource_pool

data.huaweicloud_modelarts_resource_flavors
    └── huaweicloud_modelarts_resource_pool

data.huaweicloud_modelarts_notebook_flavors
    ├── data.huaweicloud_modelarts_notebook_images
    └── huaweicloud_modelarts_notebook

data.huaweicloud_modelarts_notebook_images
    └── huaweicloud_modelarts_notebook

data.huaweicloud_modelartsv2_resource_pools
    └── huaweicloud_modelarts_notebook

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
    ├── huaweicloud_modelarts_network
    └── huaweicloud_modelarts_notebook

huaweicloud_modelarts_network
    ├── huaweicloud_modelarts_resource_pool
    └── huaweicloud_modelarts_notebook

huaweicloud_modelarts_workspace
    └── huaweicloud_modelarts_resource_pool

huaweicloud_modelarts_resource_pool
    └── huaweicloud_modelarts_notebook

huaweicloud_kps_keypair
    └── huaweicloud_modelarts_notebook

huaweicloud_modelarts_notebook
    └── huaweicloud_modelarts_notebook_mount_storage
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

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create vpc network:

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

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create vpc subnet:

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

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create security group:

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

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create security group ingress tcp rule:

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

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create security group ingress udp rule:

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

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create sfs turbo file system:

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

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create modelarts network:

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
  cidr = var.network_cidr

  sfs_turbos {
    name = huaweicloud_sfs_turbo.test.name
    id   = huaweicloud_sfs_turbo.test.id
  }
}
```

**Parameter Description**:
- **name**: ModelArts network name, assigned by referencing the input variable network_name
- **cidr**: ModelArts network CIDR block, assigned by referencing the input variable network_cidr, default value is "10.168.0.0/16"
- **sfs_turbos.name**: Associated SFS Turbo name, referencing the name of the previously created SFS Turbo resource
- **sfs_turbos.id**: Associated SFS Turbo ID, referencing the ID of the previously created SFS Turbo resource

### 10. Create ModelArts Workspace (Optional)

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create modelarts workspace:

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

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query modelarts dedicated resource pool flavors:

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

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create modelarts dedicated resource pool:

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

### 13. Query ModelArts Notebook Flavors

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query modelarts notebook flavors:

```hcl
variable "notebook_flavor_id" {
  type        = string
  default     = ""
  nullable    = false
  description = "The flavor ID of the notebook. If empty, the first available dedicated flavor is used"
}

variable "notebook_flavor_category" {
  type        = string
  default     = "CPU"
  description = "The processor type of the notebook flavor"
}

# Get ModelArts Notebook flavor information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create Notebook instances
data "huaweicloud_modelarts_notebook_flavors" "test" {
  count = var.notebook_flavor_id != "" ? 0 : 1

  type     = "DEDICATED"
  category = var.notebook_flavor_category
}

locals {
  available_notebook_flavors = [for o in try(data.huaweicloud_modelarts_notebook_flavors.test[0].flavors, []) : o if !o.sold_out]
}
```

**Parameter Description**:
- **count**: Number of data sources to create, executes flavor query only when notebook_flavor_id is empty
- **type**: Notebook type, set to "DEDICATED" for dedicated resource pool Notebooks
- **category**: Processor type, assigned by referencing the input variable notebook_flavor_category, default value is "CPU"
- **available_notebook_flavors**: Local variable, filters Notebook flavors that are not sold out

### 14. Query ModelArts Notebook Images

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query modelarts notebook images:

```hcl
variable "notebook_image_id" {
  type        = string
  default     = ""
  nullable    = false
  description = "The image ID of the notebook."
}

variable "notebook_image_type" {
  type        = string
  default     = "BUILD_IN"
  description = "The type of the notebook image"
}

# Get ModelArts Notebook image information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create Notebook instances
data "huaweicloud_modelarts_notebook_images" "test" {
  count = var.notebook_image_id != "" ? 0 : 1

  type     = var.notebook_image_type
  cpu_arch = try(data.huaweicloud_modelarts_notebook_flavors.test[0].flavors[0].arch, "x86_64")
}

locals {
  available_notebook_images = [
    for o in try(data.huaweicloud_modelarts_notebook_images.test[0].images, []) : o if contains(o.resource_categories, var.notebook_flavor_category) && o.status == "ACTIVE" && contains(o.dev_services, "NOTEBOOK")
  ]
}
```

**Parameter Description**:
- **count**: Number of data sources to create, executes image query only when notebook_image_id is empty
- **type**: Image type, assigned by referencing the input variable notebook_image_type, default value is "BUILD_IN"
- **cpu_arch**: CPU architecture, assigned based on the arch of the first flavor returned by the Notebook flavor query data source, default value is "x86_64"
- **available_notebook_images**: Local variable, filters images with ACTIVE status that support the NOTEBOOK service

### 15. Create KPS Key Pair (Optional)

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create kps key pair:

```hcl
variable "notebook_key_pair_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The key pair name for remote SSH access"

  validation {
    condition     = length(var.allowed_access_ips) == 0 || var.notebook_key_pair_name != "" || var.keypair_name != ""
    error_message = "When allowed_access_ips is configured, either notebook_key_pair_name or keypair_name must be specified."
  }
}

variable "allowed_access_ips" {
  type        = list(string)
  default     = []
  nullable    = false
  description = "The public IP addresses that are allowed for remote SSH access"
}

variable "keypair_name" {
  type        = string
  default     = ""
  nullable    = false
  description = "The name of the KPS key pair"
}

# Create a KPS key pair resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_kps_keypair" "test" {
  count = var.notebook_key_pair_name == "" && length(var.allowed_access_ips) > 0 ? 1 : 0

  name = var.keypair_name
}
```

**Parameter Description**:
- **count**: Number of resources to create, creates a key pair only when notebook_key_pair_name is empty and allowed_access_ips is not empty
- **name**: Key pair name, assigned by referencing the input variable keypair_name

> When allowed_access_ips is configured, either notebook_key_pair_name or keypair_name must be specified.

### 16. Query ModelArts V2 Resource Pool Information

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to query modelarts v2 resource pool information:

```hcl
# Get ModelArts V2 resource pool information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to obtain the Notebook workspace ID
data "huaweicloud_modelartsv2_resource_pools" "test" {}

locals {
  resource_pool = try([for pool in data.huaweicloud_modelartsv2_resource_pools.test.resource_pools : pool if pool.metadata[0].name ==
  huaweicloud_modelarts_resource_pool.test.id][0], {})
}
```

**Parameter Description**:
- This data source requires no additional parameters and automatically queries all V2 resource pools in the current region
- **resource_pool**: Local variable, matches the corresponding V2 resource pool information based on the dedicated resource pool ID

### 17. Create ModelArts Notebook Instance

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create modelarts notebook instance:

```hcl
variable "notebook_name" {
  type        = string
  description = "The name of the notebook"
}

variable "notebook_description" {
  type        = string
  default     = null
  description = "The description of the notebook"
}

variable "notebook_tags" {
  type        = map(string)
  default     = {}
  description = "The tags of the notebook"
}

# Create a ModelArts Notebook instance resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_modelarts_notebook" "test" {
  name               = var.notebook_name
  flavor_id          = var.notebook_flavor_id != "" ? var.notebook_flavor_id : try(local.available_notebook_flavors[0].id, null)
  image_id           = var.notebook_image_id != "" ? var.notebook_image_id : try(local.available_notebook_images[0].id, null)
  description        = var.notebook_description
  pool_id            = huaweicloud_modelarts_resource_pool.test.id
  workspace_id       = try(local.resource_pool.metadata[0].labels["os.modelarts/workspace.id"], null)
  key_pair           = var.notebook_key_pair_name != "" ? var.notebook_key_pair_name : try(huaweicloud_kps_keypair.test[0].name, null)
  allowed_access_ips = length(var.allowed_access_ips) > 0 ? var.allowed_access_ips : null

  volume {
    type      = "EFS"
    ownership = "DEDICATED"
    uri       = format("%s:/", try(huaweicloud_modelarts_network.test.sfs_turbos[0].uri, null))
    id        = huaweicloud_sfs_turbo.test.id
  }

  tags = var.notebook_tags
}
```

**Parameter Description**:
- **name**: Notebook name, assigned by referencing the input variable notebook_name
- **flavor_id**: Notebook flavor ID, uses notebook_flavor_id when it is not empty, otherwise selects the first from the available flavor list
- **image_id**: Notebook image ID, uses notebook_image_id when it is not empty, otherwise selects the first from the available image list
- **description**: Notebook description, assigned by referencing the input variable notebook_description
- **pool_id**: Dedicated resource pool ID, referencing the ID of the previously created dedicated resource pool resource
- **workspace_id**: Workspace ID, obtained from metadata.labels of the V2 resource pool information
- **key_pair**: Key pair name, uses notebook_key_pair_name when it is not empty, otherwise references the created KPS key pair resource name
- **allowed_access_ips**: List of public IPs allowed for remote SSH access, assigned by referencing the input variable allowed_access_ips
- **volume**: Storage volume configuration, type EFS, ownership DEDICATED, associated with SFS Turbo
- **tags**: Notebook tags, assigned by referencing the input variable notebook_tags, default value is an empty map

### 18. Create ModelArts Notebook Mount Storage (Optional)

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create modelarts notebook mount storage:

```hcl
variable "notebook_mount_storage_path" {
  type        = string
  default     = ""
  nullable    = false
  description = "The OBS path of Parallel File System (PFS) or its folders to mount"
}

variable "notebook_mount_storage_local_directory" {
  type        = string
  default     = ""
  nullable    = false
  description = "The local mount directory for the OBS storage"
}

# Create a ModelArts Notebook mount storage resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_modelarts_notebook_mount_storage" "test" {
  count = var.notebook_mount_storage_path != "" && var.notebook_mount_storage_local_directory != "" ? 1 : 0

  notebook_id           = huaweicloud_modelarts_notebook.test.id
  storage_path          = var.notebook_mount_storage_path
  local_mount_directory = var.notebook_mount_storage_local_directory
}
```

**Parameter Description**:
- **count**: Number of resources to create, created only when both notebook_mount_storage_path and notebook_mount_storage_local_directory are not empty
- **notebook_id**: Notebook ID, referencing the ID of the previously created Notebook resource
- **storage_path**: OBS path of Parallel File System (PFS) or its folders, assigned by referencing the input variable notebook_mount_storage_path
- **local_mount_directory**: Local mount directory, assigned by referencing the input variable notebook_mount_storage_local_directory

### 19. Preset Input Parameters for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration values. These input parameters need to be entered manually during subsequent deployment.
Terraform also provides a method to preset these configurations through a `terraform.tfvars` file to avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC network configuration
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"

# SFS Turbo configuration
turbo_name = "tf_test_sfs_turbo"

# ModelArts network and resource pool configuration
network_name       = "tf-test-network"
resource_pool_name = "tf-test-resource-pool"

# Notebook configuration
notebook_name        = "tf_test_notebook"
notebook_description = "Created by Terraform script"

notebook_tags = {
  owner = "terraform"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the contents when executing terraform commands; other names require adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read variable values from this file

In addition to using a `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="notebook_name=my-notebook"`
2. Environment variables: `export TF_VAR_notebook_name=my-notebook`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform uses variable values in the following priority: command line parameters > variable files > environment variables > default values.

### 20. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the dedicated resource pool and Notebook instance
4. Run `terraform show` to view details of the created dedicated resource pool and Notebook instance

## Reference Information

- [Huawei Cloud ModelArts Product Documentation](https://support.huaweicloud.com/modelarts/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ModelArts Notebook with Dedicated Resource Pool](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/modelarts/notebook-with-dedicated-resource-pool)
