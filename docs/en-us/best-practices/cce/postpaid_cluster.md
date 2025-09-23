# Deploy Postpaid Cluster

## Application Scenario

Huawei Cloud Cloud Container Engine (CCE) is a high-reliability, high-performance enterprise-grade container management service that supports Kubernetes community native applications and tools. By deploying postpaid CCE clusters, enterprises can quickly build containerized application environments, implement microservices architecture deployment, and bill based on actual usage, effectively controlling costs. This best practice will introduce how to use Terraform to automatically deploy a postpaid CCE cluster, including the creation of VPC, subnet, CCE cluster, and nodes.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Image Query Data Source (data.huaweicloud_images_image)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_image)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Elastic Public IP Resource (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [Key Pair Resource (huaweicloud_compute_keypair)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_keypair)
- [CCE Cluster Resource (huaweicloud_cce_cluster)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_cluster)
- [CCE Node Resource (huaweicloud_cce_node)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_node)
- [Compute Instance Resource (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [CCE Node Attach Resource (huaweicloud_cce_node_attach)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_node_attach)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── huaweicloud_cce_node
    └── huaweicloud_compute_instance

data.huaweicloud_images_image
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_cce_cluster
        │   ├── huaweicloud_cce_node
        │   └── huaweicloud_cce_node_attach
        └── huaweicloud_compute_instance

huaweicloud_vpc_eip
    └── huaweicloud_cce_cluster

huaweicloud_compute_keypair
    ├── huaweicloud_cce_node
    ├── huaweicloud_compute_instance
    └── huaweicloud_cce_node_attach

huaweicloud_compute_instance
    └── huaweicloud_cce_node_attach
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zones Required for CCE Cluster Resource Creation via Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the result of which will be used to create the CCE cluster:

```hcl
# Get all availability zone information under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create the CCE cluster
data "huaweicloud_availability_zones" "myaz" {}
```

**Parameter Description**:
This data source requires no additional parameters and defaults to querying all available availability zone information in the current region.

### 3. Query Images Required for CCE Cluster Resource Creation via Data Source

Add the following script to the TF file to instruct Terraform to query images that meet the conditions:

```hcl
variable "image_name" {
  description = "ECS image name"
  type        = string
}

# Get all image information that meets specific conditions under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to create the CCE cluster
data "huaweicloud_images_image" "myimage" {
  name        = var.image_name
  most_recent = true
}
```

**Parameter Description**:
- **name**: Image name, assigned by referencing the input variable image_name
- **most_recent**: Whether to use the latest version of the image, set to true to use the latest version

### 4. Create VPC Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

```hcl
variable "vpc_name" {
  description = "VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
}

# Create a VPC resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to deploy the CCE cluster
resource "huaweicloud_vpc" "myvpc" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing the input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing the input variable vpc_cidr

### 5. Create VPC Subnet Resource

Add the following script to the TF file to instruct Terraform to create a VPC subnet resource:

```hcl
variable "subnet_name" {
  description = "Subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "Subnet CIDR block"
  type        = string
}

variable "subnet_gateway" {
  description = "Subnet gateway address"
  type        = string
}

variable "primary_dns" {
  description = "Primary DNS server for the subnet"
  type        = string
}

variable "secondary_dns" {
  description = "Secondary DNS server for the subnet"
  type        = string
}

# Create a VPC subnet resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to deploy the CCE cluster
resource "huaweicloud_vpc_subnet" "mysubnet" {
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway

  # dns is required for cce node installing
  primary_dns   = var.primary_dns
  secondary_dns = var.secondary_dns
  vpc_id        = huaweicloud_vpc.myvpc.id
}
```

**Parameter Description**:
- **name**: Subnet name, assigned by referencing the input variable subnet_name
- **cidr**: Subnet CIDR block, assigned by referencing the input variable subnet_cidr
- **gateway_ip**: Gateway IP address, assigned by referencing the input variable subnet_gateway
- **primary_dns**: Primary DNS server address, assigned by referencing the input variable primary_dns
- **secondary_dns**: Secondary DNS server address, assigned by referencing the input variable secondary_dns
- **vpc_id**: VPC ID to which the subnet belongs, referencing the ID of the previously created VPC resource

### 6. Create Elastic Public IP Resource

Add the following script to the TF file to instruct Terraform to create an elastic public IP resource:

```hcl
variable "bandwidth_name" {
  description = "Bandwidth name"
  type        = string
}

# Create an elastic public IP resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to deploy the CCE cluster
resource "huaweicloud_vpc_eip" "myeip" {
  publicip {
    type = "5_bgp"
  }
  bandwidth {
    name        = var.bandwidth_name
    size        = 8
    share_type  = "PER"
    charge_mode = "traffic"
  }
}
```

**Parameter Description**:
- **type**: Elastic public IP type, set to "5_bgp"
- **name**: Bandwidth name, assigned by referencing the input variable bandwidth_name
- **size**: Bandwidth size (Mbit/s), set to 8Mbps
- **share_type**: Bandwidth sharing type, set to "PER" for dedicated
- **charge_mode**: Billing mode, set to "traffic" for pay-per-use

### 7. Create Key Pair Resource

Add the following script to the TF file to instruct Terraform to create a key pair resource:

```hcl
variable "key_pair_name" {
  description = "Key pair name"
  type        = string
}

# Create a key pair resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block), used to deploy the CCE cluster
resource "huaweicloud_compute_keypair" "mykeypair" {
  name = var.key_pair_name
}
```

**Parameter Description**:
- **name**: Key pair name, assigned by referencing the input variable key_pair_name

### 8. Create CCE Cluster Resource

Add the following script to the TF file to instruct Terraform to create a CCE cluster resource:

```hcl
variable "cce_cluster_name" {
  description = "CCE cluster name"
  type        = string
}

variable "cce_cluster_flavor" {
  description = "CCE cluster flavor"
  type        = string
}

# Create a CCE cluster resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_cce_cluster" "mycce" {
  name                   = var.cce_cluster_name
  flavor_id              = var.cce_cluster_flavor
  vpc_id                 = huaweicloud_vpc.myvpc.id
  subnet_id              = huaweicloud_vpc_subnet.mysubnet.id
  container_network_type = "overlay_l2"
  eip                    = huaweicloud_vpc_eip.myeip.address
}
```

**Parameter Description**:
- **name**: Cluster name, assigned by referencing the input variable cce_cluster_name
- **flavor_id**: Cluster flavor, assigned by referencing the input variable cce_cluster_flavor
- **vpc_id**: VPC ID, referencing the ID of the previously created VPC resource
- **subnet_id**: Subnet ID, referencing the ID of the previously created subnet resource
- **container_network_type**: Container network type, set to "overlay_l2"
- **eip**: Elastic public IP address, referencing the address of the previously created elastic public IP resource

### 9. Create CCE Node Resource

Add the following script to the TF file to instruct Terraform to create a CCE node resource:

```hcl
variable "node_name" {
  description = "CCE node name"
  type        = string
}

variable "node_flavor" {
  description = "CCE node flavor"
  type        = string
}

variable "root_volume_size" {
  description = "System disk size"
  type        = number
}

variable "root_volume_type" {
  description = "System disk type"
  type        = string
}

variable "data_volume_size" {
  description = "Data disk size"
  type        = number
}

variable "data_volume_type" {
  description = "Data disk type"
  type        = string
}

# Create a CCE node resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_cce_node" "mynode" {
  cluster_id        = huaweicloud_cce_cluster.mycce.id
  name              = var.node_name
  flavor_id         = var.node_flavor
  availability_zone = data.huaweicloud_availability_zones.myaz.names[0]
  key_pair          = huaweicloud_compute_keypair.mykeypair.name

  root_volume {
    size       = var.root_volume_size
    volumetype = var.root_volume_type
  }
  data_volumes {
    size       = var.data_volume_size
    volumetype = var.data_volume_type
  }
}
```

**Parameter Description**:
- **cluster_id**: CCE cluster ID, referencing the ID of the previously created CCE cluster resource
- **name**: Node name, assigned by referencing the input variable node_name
- **flavor_id**: Node flavor, assigned by referencing the input variable node_flavor
- **availability_zone**: Availability zone, using the first availability zone from the availability zone list query data source
- **key_pair**: Key pair name, referencing the name of the previously created key pair resource
- **root_volume.size**: System disk size, assigned by referencing the input variable root_volume_size
- **root_volume.volumetype**: System disk type, assigned by referencing the input variable root_volume_type
- **data_volumes.size**: Data disk size, assigned by referencing the input variable data_volume_size
- **data_volumes.volumetype**: Data disk type, assigned by referencing the input variable data_volume_type

### 10. Create Compute Instance Resource

Add the following script to the TF file to instruct Terraform to create a compute instance resource:

```hcl
variable "ecs_name" {
  description = "ECS instance name"
  type        = string
}

variable "ecs_flavor" {
  description = "ECS instance flavor"
  type        = string
}

# Create a compute instance resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_compute_instance" "myecs" {
  name                        = var.ecs_name
  image_id                    = data.huaweicloud_images_image.myimage.id
  flavor_id                   = var.ecs_flavor
  availability_zone           = data.huaweicloud_availability_zones.myaz.names[0]
  key_pair                    = huaweicloud_compute_keypair.mykeypair.name
  delete_disks_on_termination = true

  system_disk_type = var.root_volume_type
  system_disk_size = var.root_volume_size

  data_disks {
    type = var.data_volume_type
    size = var.data_volume_size
  }

  network {
    uuid = huaweicloud_vpc_subnet.mysubnet.id
  }
}
```

**Parameter Description**:
- **name**: Instance name, assigned by referencing the input variable ecs_name
- **image_id**: Image ID, using the ID from the image query data source
- **flavor_id**: Instance flavor, assigned by referencing the input variable ecs_flavor
- **availability_zone**: Availability zone, using the first availability zone from the availability zone list query data source
- **key_pair**: Key pair name, referencing the name of the previously created key pair resource
- **delete_disks_on_termination**: Whether to delete disks when instance is deleted, set to true
- **system_disk_type**: System disk type, assigned by referencing the input variable root_volume_type
- **system_disk_size**: System disk size, assigned by referencing the input variable root_volume_size
- **data_disks.type**: Data disk type, assigned by referencing the input variable data_volume_type
- **data_disks.size**: Data disk size, assigned by referencing the input variable data_volume_size
- **network.uuid**: Network ID, referencing the ID of the previously created subnet resource

### 11. Create CCE Node Attach Resource

Add the following script to the TF file to instruct Terraform to create a CCE node attach resource:

```hcl
variable "os" {
  description = "Operating system type"
  type        = string
}

# Create a CCE node attach resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_cce_node_attach" "test" {
  cluster_id = huaweicloud_cce_cluster.mycce.id
  server_id  = huaweicloud_compute_instance.myecs.id
  key_pair   = huaweicloud_compute_keypair.mykeypair.name
  os         = var.os
}
```

**Parameter Description**:
- **cluster_id**: CCE cluster ID, referencing the ID of the previously created CCE cluster resource
- **server_id**: ECS instance ID, referencing the ID of the previously created compute instance resource
- **key_pair**: Key pair name, referencing the name of the previously created key pair resource
- **os**: Operating system type, assigned by referencing the input variable os

### 12. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# Image configuration
image_name = "Ubuntu 18.04 server 64bit"

# VPC configuration
vpc_name = "tf_test_vpc"
vpc_cidr = "192.168.0.0/16"

# Subnet configuration
subnet_name = "tf_test_subnet"
subnet_cidr = "192.168.1.0/24"
subnet_gateway = "192.168.1.1"
primary_dns = "8.8.8.8"
secondary_dns = "8.8.4.4"

# Elastic public IP configuration
bandwidth_name = "tf_test_bandwidth"

# Key pair configuration
key_pair_name = "tf_test_keypair"

# CCE cluster configuration
cce_cluster_name = "tf_test_cluster"
cce_cluster_flavor = "cce.s1.small"

# CCE node configuration
node_name = "tf_test_node"
node_flavor = "s6.large.2"
root_volume_size = 40
root_volume_type = "SSD"
data_volume_size = 100
data_volume_type = "SSD"

# ECS instance configuration
ecs_name = "tf_test_ecs"
ecs_flavor = "s6.large.2"

# Operating system configuration
os = "EulerOS 2.5"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 13. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the CCE cluster
4. Run `terraform show` to view the details of the created CCE cluster

## Reference Information

- [Huawei Cloud CCE Product Documentation](https://support.huaweicloud.com/cce/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE Cluster Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cce)
