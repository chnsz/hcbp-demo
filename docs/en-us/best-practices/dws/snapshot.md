# Deploy Snapshot

## Application Scenario

Data Warehouse Service (DWS) is an online analytical processing (OLAP) enterprise-level data warehouse service provided by Huawei Cloud, delivering high-performance, highly reliable, and easily scalable data warehouse capabilities for massive data analysis scenarios. A snapshot is an important data protection method for DWS clusters. It can back up cluster data before cluster upgrades, data misoperations, or business changes, and restore the cluster to the state when the snapshot was created when needed.

This best practice will introduce how to use Terraform to automatically deploy a DWS snapshot, including VPC, subnet, and security group creation, DWS cluster creation, and snapshot creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DWS Cluster Flavors (data.huaweicloud_dws_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dws_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DWS Cluster (huaweicloud_dws_cluster)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_cluster)
- [DWS Snapshot (huaweicloud_dws_snapshot)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_snapshot)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dws_cluster

data.huaweicloud_dws_flavors
    └── huaweicloud_dws_cluster

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dws_cluster
            └── huaweicloud_dws_snapshot

huaweicloud_networking_secgroup
    └── huaweicloud_dws_cluster
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query the Availability Zone List

Add the following script in the TF file (such as main.tf) to query the availability zones available for the DWS cluster:

```hcl
# Query the availability zone list in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone of the DWS cluster"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:

- **count**: Queries the availability zone list when the input variable availability_zone is empty; otherwise, uses the availability zone specified by the user

### 3. Create a Virtual Private Cloud

Add the following script in the TF file (such as main.tf) to create a virtual private cloud:

```hcl
# Create a virtual private cloud in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:

- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; no enterprise project is specified when it is empty

### 4. Create a Virtual Private Cloud Subnet

Add the following script in the TF file (such as main.tf) to create a virtual private cloud subnet:

```hcl
# Create a virtual private cloud subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:

- **vpc_id**: Assigned by referencing the ID of the resource huaweicloud_vpc.test
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; the subnet CIDR is automatically divided based on the VPC CIDR when it is empty
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip; the gateway address is automatically calculated based on the subnet CIDR when it is empty

### 5. Create a Security Group

Add the following script in the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

variable "security_group_delete_default_rules" {
  description = "Whether to delete the default rules of the security group"
  type        = bool
  default     = true
}

resource "huaweicloud_networking_secgroup" "test" {
  name                  = var.security_group_name
  delete_default_rules  = var.security_group_delete_default_rules
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:

- **name**: Assigned by referencing the input variable security_group_name
- **delete_default_rules**: Assigned by referencing the input variable security_group_delete_default_rules, used to control whether to delete the default rules of the security group
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; no enterprise project is specified when it is empty

### 6. Query DWS Cluster Flavors

Add the following script in the TF file (such as main.tf) to query DWS cluster flavors:

```hcl
# Query DWS cluster flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "cluster_node_type" {
  description = "The flavor of the DWS cluster node"
  type        = string
  default     = ""
  nullable    = false
}

variable "cluster_version" {
  description = "The version of the DWS cluster"
  type        = string
  default     = ""
  nullable    = false
}

variable "cluster_vcpus" {
  description = "The vcpus of the DWS cluster"
  type        = number
  default     = 4
}

variable "cluster_memory" {
  description = "The memory of the DWS cluster"
  type        = number
  default     = 32
}

variable "cluster_datastore_type" {
  description = "The datastore type of the DWS cluster"
  type        = string
  default     = "dws"
}

data "huaweicloud_dws_flavors" "test" {
  count = var.cluster_node_type == "" || var.cluster_version == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  vcpus             = var.cluster_vcpus
  memory            = var.cluster_memory
  datastore_type    = var.cluster_datastore_type
}
```

**Parameter Description**:

- **count**: Queries DWS cluster flavors when the input variable cluster_node_type or cluster_version is empty
- **availability_zone**: Assigned by referencing the input variable availability_zone or the availability zone list data source
- **vcpus**: Assigned by referencing the input variable cluster_vcpus
- **memory**: Assigned by referencing the input variable cluster_memory
- **datastore_type**: Assigned by referencing the input variable cluster_datastore_type

### 7. Create a DWS Cluster

Add the following script in the TF file (such as main.tf) to create a DWS cluster:

```hcl
# Create a DWS cluster in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "cluster_name" {
  description = "The name of the DWS cluster"
  type        = string
}

variable "cluster_number_of_node" {
  description = "The number of nodes in the DWS cluster"
  type        = number
  default     = 3
}

variable "cluster_number_of_cn" {
  description = "The number of CN nodes in the DWS cluster"
  type        = number
  default     = 3
}

variable "cluster_admin_user_name" {
  description = "The administrator username of the DWS cluster"
  type        = string
}

variable "cluster_admin_user_pwd" {
  description = "The administrator password of the DWS cluster"
  type        = string
  sensitive   = true
}

variable "cluster_volume_type" {
  description = "The volume type of the DWS cluster"
  type        = string
  default     = "SSD"
}

variable "cluster_volume_capacity" {
  description = "The volume capacity of the DWS cluster in GB"
  type        = string
  default     = "100"
}

resource "huaweicloud_dws_cluster" "test" {
  name                  = var.cluster_name
  node_type             = var.cluster_node_type != "" ? var.cluster_node_type : try(data.huaweicloud_dws_flavors.test[0].flavors[0].flavor_id, null)
  number_of_node        = var.cluster_number_of_node
  number_of_cn          = var.cluster_number_of_cn
  version               = var.cluster_version != "" ? var.cluster_version : try(data.huaweicloud_dws_flavors.test[0].flavors[0].datastore_version, null)
  vpc_id                = huaweicloud_vpc.test.id
  network_id            = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  user_name             = var.cluster_admin_user_name
  user_pwd              = var.cluster_admin_user_pwd
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  volume {
    type     = var.cluster_volume_type
    capacity = var.cluster_volume_capacity
  }
}
```

**Parameter Description**:

- **name**: Assigned by referencing the input variable cluster_name
- **node_type**: Assigned by referencing the input variable cluster_node_type or the DWS cluster flavors data source
- **number_of_node**: Assigned by referencing the input variable cluster_number_of_node
- **number_of_cn**: Assigned by referencing the input variable cluster_number_of_cn
- **version**: Assigned by referencing the input variable cluster_version or the DWS cluster flavors data source
- **vpc_id**: Assigned by referencing the ID of the resource huaweicloud_vpc.test
- **network_id**: Assigned by referencing the ID of the resource huaweicloud_vpc_subnet.test
- **security_group_id**: Assigned by referencing the ID of the resource huaweicloud_networking_secgroup.test
- **availability_zone**: Assigned by referencing the input variable availability_zone or the availability zone list data source
- **user_name**: Assigned by referencing the input variable cluster_admin_user_name
- **user_pwd**: Assigned by referencing the input variable cluster_admin_user_pwd
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; no enterprise project is specified when it is empty
- **volume.type**: Assigned by referencing the input variable cluster_volume_type
- **volume.capacity**: Assigned by referencing the input variable cluster_volume_capacity

### 8. Create a DWS Snapshot

Add the following script in the TF file (such as main.tf) to create a DWS snapshot:

```hcl
# Create a DWS snapshot in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "snapshot_name" {
  description = "The name of the snapshot"
  type        = string
}

variable "snapshot_description" {
  description = "The description of the snapshot"
  type        = string
  default     = ""
}

resource "huaweicloud_dws_snapshot" "test" {
  name        = var.snapshot_name
  cluster_id  = huaweicloud_dws_cluster.test.id
  description = var.snapshot_description
}
```

**Parameter Description**:

- **name**: Assigned by referencing the input variable snapshot_name
- **cluster_id**: Assigned by referencing the ID of the resource huaweicloud_dws_cluster.test
- **description**: Assigned by referencing the input variable snapshot_description

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name                = "tf_test_dws_vpc"
vpc_cidr                = "192.168.0.0/16"
subnet_name             = "tf_test_dws_subnet"
security_group_name     = "tf_test_dws_sg"
cluster_name            = "tf_test_dws_cluster"
cluster_admin_user_name = "dbadmin"
cluster_admin_user_pwd  = "YourPassword@123"
snapshot_name           = "tf_test_dws_snapshot"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DWS snapshot
4. Run `terraform show` to view the created DWS snapshot

## Reference Information

- [Huawei Cloud Data Warehouse Service Product Documentation](https://support.huaweicloud.com/dws/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DWS Snapshot](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dws/snapshot)
