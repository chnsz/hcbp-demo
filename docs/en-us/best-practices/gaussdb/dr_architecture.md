# Deploy Disaster Recovery Architecture

## Application Scenario

GaussDB is a high-performance, highly available, and highly secure enterprise-grade distributed relational database service provided by Huawei Cloud, supporting both centralized and distributed deployment modes. For core business systems, a single-instance deployment cannot meet business continuity requirements. A disaster recovery architecture across availability zones or even regions is needed so that services can be switched over quickly when the primary instance fails, ensuring no data loss and no service interruption.

This best practice will introduce how to use Terraform to automatically deploy a GaussDB disaster recovery architecture, including the creation of VPCs, subnets, and security groups in both the primary and DR regions, cross-region network connectivity through Cloud Connect (CC), creation of primary and standby GaussDB instances, DR configuration reset, and establishment of the DR relationship.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [IAM Projects (data.huaweicloud_identity_projects)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Cloud Connection (huaweicloud_cc_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_connection)
- [Cloud Connection Network Instance (huaweicloud_cc_network_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_network_instance)
- [Cloud Connection Bandwidth Package (huaweicloud_cc_bandwidth_package)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_bandwidth_package)
- [Cloud Connection Inter-Region Bandwidth (huaweicloud_cc_inter_region_bandwidth)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_inter_region_bandwidth)
- [Random Password (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB Instance (huaweicloud_gaussdb_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB DR Configuration Reset (huaweicloud_gaussdb_dr_configuration_reset)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_dr_configuration_reset)
- [GaussDB DR Relationship (huaweicloud_gaussdb_dr_relationship)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_dr_relationship)

### Resource/Data Source Dependencies

```
data.huaweicloud_identity_projects
    ├── huaweicloud_cc_network_instance.primary
    └── huaweicloud_cc_network_instance.dr

huaweicloud_vpc.primary
    ├── huaweicloud_vpc_subnet.primary
    ├── huaweicloud_networking_secgroup.primary
    └── huaweicloud_cc_network_instance.primary

huaweicloud_vpc.dr
    ├── huaweicloud_vpc_subnet.dr
    ├── huaweicloud_networking_secgroup.dr
    └── huaweicloud_cc_network_instance.dr

huaweicloud_cc_connection.dr
    ├── huaweicloud_cc_network_instance.primary
    ├── huaweicloud_cc_network_instance.dr
    ├── huaweicloud_cc_bandwidth_package.dr
    └── huaweicloud_cc_inter_region_bandwidth.dr

huaweicloud_gaussdb_instance.primary
    ├── huaweicloud_gaussdb_dr_configuration_reset.primary
    └── huaweicloud_gaussdb_dr_relationship.test

huaweicloud_gaussdb_instance.dr
    └── huaweicloud_gaussdb_dr_configuration_reset.dr
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Create Primary Region Network Resources

Add the following script in the TF file (such as main.tf) to create the VPC, subnet, security group, and security group rules in the primary region:

```hcl
# Create the VPC resource in the primary region in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "primary_vpc_name" {
  description = "The VPC name in the primary region"
  type        = string
}

variable "primary_vpc_cidr" {
  description = "The CIDR block of the VPC in the primary region"
  type        = string
  default     = "172.16.0.0/16"
}

variable "primary_subnet_names" {
  description = "The subnet names for each AZ in the primary region"
  type        = list(string)
}

variable "primary_availability_zones" {
  description = "The availability zones in the primary region"
  type        = list(string)
}

variable "primary_security_group_name" {
  description = "The security group name in the primary region"
  type        = string
}

variable "secgroup_rules" {
  description = "The security group ingress rules"
  type        = list(object({
    ports  = string
    source = string
  }))
  default     = [
    { ports = "5432-5532", source = "local" },
    { ports = "5432-5532", source = "remote" },
    { ports = "20050",     source = "local" },
    { ports = "20050",     source = "remote" },
    { ports = "5000-5001", source = "local" },
    { ports = "5000-5001", source = "remote" },
    { ports = "2379-2380", source = "local" },
    { ports = "2379-2380", source = "remote" },
    { ports = "6000",      source = "local" },
    { ports = "6000",      source = "remote" },
    { ports = "6500",      source = "local" },
    { ports = "6500",      source = "remote" },
    { ports = "12016",     source = "local" },
    { ports = "12016",     source = "remote" },
  ]
}

resource "huaweicloud_vpc" "primary" {
  name = var.primary_vpc_name
  cidr = var.primary_vpc_cidr
}

resource "huaweicloud_vpc_subnet" "primary" {
  count = length(var.primary_subnet_names)

  vpc_id            = huaweicloud_vpc.primary.id
  name              = try(var.primary_subnet_names[count.index], null)
  cidr              = cidrsubnet(huaweicloud_vpc.primary.cidr, 8, count.index)
  gateway_ip        = cidrhost(cidrsubnet(huaweicloud_vpc.primary.cidr, 8, count.index), 1)
  availability_zone = try(var.primary_availability_zones[count.index], null)
}

resource "huaweicloud_networking_secgroup" "primary" {
  name                 = var.primary_security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "primary" {
  count = length(var.secgroup_rules)

  security_group_id = huaweicloud_networking_secgroup.primary.id
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = var.secgroup_rules[count.index].source == "local" ? huaweicloud_vpc.primary.cidr : huaweicloud_vpc.dr.cidr
  ports             = try(var.secgroup_rules[count.index].ports, null)
  protocol          = "tcp"
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable primary_vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable primary_vpc_cidr
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing `huaweicloud_vpc.primary.id`
- **availability_zone**: The availability zone where the subnet is located, assigned by referencing the input variable primary_availability_zones
- **security_group_id**: The ID of the security group to which the rule belongs, referencing `huaweicloud_networking_secgroup.primary.id`
- **remote_ip_prefix**: The remote IP address range of the security group rule, referencing the VPC CIDR of the primary or DR region based on the rule source
- **ports**: The port range opened by the security group rule, assigned by referencing the input variable secgroup_rules

### 3. Create DR Region Network Resources

Add the following script in the TF file (such as main.tf) to create the VPC, subnet, security group, and security group rules in the DR region:

```hcl
# Create the VPC resource in the DR region in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dr_vpc_name" {
  description = "The VPC name in the DR region"
  type        = string
}

variable "dr_vpc_cidr" {
  description = "The CIDR block of the VPC in the DR region"
  type        = string
  default     = "172.17.0.0/16"
}

variable "dr_subnet_name" {
  description = "The subnet name in the DR region"
  type        = string
}

variable "dr_availability_zone" {
  description = "The availability zone in the DR region"
  type        = string
}

variable "dr_security_group_name" {
  description = "The security group name in the DR region"
  type        = string
}

resource "huaweicloud_vpc" "dr" {
  provider = huaweicloud.dr

  name = var.dr_vpc_name
  cidr = var.dr_vpc_cidr
}

resource "huaweicloud_vpc_subnet" "dr" {
  provider = huaweicloud.dr

  vpc_id            = huaweicloud_vpc.dr.id
  name              = var.dr_subnet_name
  cidr              = cidrsubnet(huaweicloud_vpc.dr.cidr, 8, 0)
  gateway_ip        = cidrhost(cidrsubnet(huaweicloud_vpc.dr.cidr, 8, 0), 1)
  availability_zone = var.dr_availability_zone
}

resource "huaweicloud_networking_secgroup" "dr" {
  provider = huaweicloud.dr

  name                 = var.dr_security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "dr" {
  provider = huaweicloud.dr

  count = length(var.secgroup_rules)

  security_group_id = huaweicloud_networking_secgroup.dr.id
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = var.secgroup_rules[count.index].source == "local" ? huaweicloud_vpc.dr.cidr : huaweicloud_vpc.primary.cidr
  ports             = try(var.secgroup_rules[count.index].ports, null)
  protocol          = "tcp"
}
```

**Parameter Description**:
- **provider**: Specifies the provider with the alias `huaweicloud.dr` to create resources in the DR region
- **name**: The VPC name, assigned by referencing the input variable dr_vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable dr_vpc_cidr
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing `huaweicloud_vpc.dr.id`
- **availability_zone**: The availability zone where the subnet is located, assigned by referencing the input variable dr_availability_zone
- **security_group_id**: The ID of the security group to which the rule belongs, referencing `huaweicloud_networking_secgroup.dr.id`
- **remote_ip_prefix**: The remote IP address range of the security group rule, referencing the VPC CIDR of the DR or primary region based on the rule source

### 4. Create Cloud Connection Network Instances

Add the following script in the TF file (such as main.tf) to query IAM projects and create the cloud connection and network instances:

```hcl
# Query IAM project information in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_identity_projects" "primary" {
  name = var.region_name
}

data "huaweicloud_identity_projects" "dr" {
  provider = huaweicloud.dr

  name = var.dr_region_name
}

# Create the cloud connection resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_cc_connection" "dr" {
  name        = "${var.primary_vpc_name}_cc"
  description = "Cloud connection for GaussDB cross-region DR"
}

resource "huaweicloud_cc_network_instance" "primary" {
  cloud_connection_id = huaweicloud_cc_connection.dr.id
  instance_id         = huaweicloud_vpc.primary.id
  project_id          = try(data.huaweicloud_identity_projects.primary.projects[0].id, null)
  region_id           = var.region_name
  type                = "vpc"
  cidrs               = [huaweicloud_vpc.primary.cidr]
}

resource "huaweicloud_cc_network_instance" "dr" {
  provider = huaweicloud.dr

  cloud_connection_id = huaweicloud_cc_connection.dr.id
  instance_id         = huaweicloud_vpc.dr.id
  project_id          = try(data.huaweicloud_identity_projects.dr.projects[0].id, null)
  region_id           = var.dr_region_name
  type                = "vpc"
  cidrs               = [huaweicloud_vpc.dr.cidr]
}
```

**Parameter Description**:
- **name**: The IAM project name, assigned by referencing the input variable region_name or dr_region_name
- **cloud_connection_id**: The ID of the cloud connection to which the network instance belongs, referencing `huaweicloud_cc_connection.dr.id`
- **instance_id**: The ID of the VPC associated with the network instance, referencing `huaweicloud_vpc.primary.id` or `huaweicloud_vpc.dr.id`
- **project_id**: The ID of the IAM project to which the network instance belongs, referencing the data source query result
- **region_id**: The region where the network instance is located, assigned by referencing the input variable region_name or dr_region_name
- **type**: The network instance type, which is `vpc` here
- **cidrs**: The CIDR blocks of the network instance, referencing the CIDR of the corresponding VPC

### 5. Create Cloud Connection Bandwidth Package and Inter-Region Bandwidth

Add the following script in the TF file (such as main.tf) to create the cloud connection bandwidth package and inter-region bandwidth:

```hcl
# Create the cloud connection bandwidth package resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "cc_bandwidth" {
  description = "The inter-region bandwidth (Mbit/s) for the Cloud Connection"
  type        = number
  default     = 10
}

resource "huaweicloud_cc_bandwidth_package" "dr" {
  name           = "${var.primary_vpc_name}_cc_bp"
  local_area_id  = "Chinese-Mainland"
  remote_area_id = "Chinese-Mainland"
  bandwidth      = var.cc_bandwidth
  billing_mode   = 3
  charge_mode    = "bandwidth"
  resource_id    = huaweicloud_cc_connection.dr.id
  resource_type  = "cloud_connection"
}

resource "huaweicloud_cc_inter_region_bandwidth" "dr" {
  cloud_connection_id  = huaweicloud_cc_connection.dr.id
  bandwidth_package_id = huaweicloud_cc_bandwidth_package.dr.id
  bandwidth            = var.cc_bandwidth
  inter_region_ids     = [var.region_name, var.dr_region_name]
}
```

**Parameter Description**:
- **name**: The bandwidth package name, generated by concatenating the primary region VPC name
- **local_area_id**: The local area of the bandwidth package, which is `Chinese-Mainland` here
- **remote_area_id**: The remote area of the bandwidth package, which is `Chinese-Mainland` here
- **bandwidth**: The bandwidth size, assigned by referencing the input variable cc_bandwidth
- **billing_mode**: The billing mode, which is `3` here
- **charge_mode**: The charging mode, which is `bandwidth` here
- **resource_id**: The ID of the cloud connection associated with the bandwidth package, referencing `huaweicloud_cc_connection.dr.id`
- **resource_type**: The resource type associated with the bandwidth package, which is `cloud_connection` here
- **cloud_connection_id**: The ID of the cloud connection to which the inter-region bandwidth belongs, referencing `huaweicloud_cc_connection.dr.id`
- **bandwidth_package_id**: The ID of the bandwidth package associated with the inter-region bandwidth, referencing `huaweicloud_cc_bandwidth_package.dr.id`
- **inter_region_ids**: The two regions connected by the inter-region bandwidth, assigned by referencing the input variables region_name and dr_region_name

### 6. Create GaussDB Instance Passwords

Add the following script in the TF file (such as main.tf) to create the random passwords required by the GaussDB instances:

```hcl
# Create the random password resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_passwords" {
  description = "The passwords for GaussDB instances (empty string to auto-generate), index 0=primary, 1=dr"
  type        = list(string)
  default     = ["", ""]
  sensitive   = true
}

resource "random_password" "instance" {
  count = length(var.instance_passwords)

  length           = 12
  special          = true
  override_special = "!@%^*-_=+"

  keepers = {
    password_needed = try(var.instance_passwords[count.index], "") == "" ? "true" : "false"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

**Parameter Description**:
- **count**: The number of random passwords, consistent with the length of the input variable instance_passwords
- **length**: The password length, which is `12` here
- **special**: Whether to include special characters, which is `true` here
- **override_special**: The set of allowed special characters
- **keepers**: The key-value pair used to trigger password regeneration, generating a new password when the corresponding element in the input variable instance_passwords is an empty string

### 7. Create Primary Region GaussDB Instance

Add the following script in the TF file (such as main.tf) to create the GaussDB instance in the primary region:

```hcl
# Create the GaussDB instance resource in the primary region in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "primary_instance_name" {
  description = "The name of the primary GaussDB instance"
  type        = string
}

variable "instance_flavor" {
  description = "The spec_code of the GaussDB instance flavor"
  type        = string
}

variable "primary_instance_availability_zones" {
  description = "The comma-separated AZ string for the primary GaussDB instance"
  type        = string
}

variable "instance_db_port" {
  description = "The database port"
  type        = number
  default     = 5432
}

variable "enterprise_project_id" {
  description = "The enterprise project ID to which the GaussDB instances belong"
  type        = string
}

variable "primary_instance_volume_type" {
  description = "The storage volume type of the primary GaussDB instance"
  type        = string
  default     = "ULTRAHIGH"
}

variable "primary_instance_volume_size" {
  description = "The storage volume size (GB) of the primary GaussDB instance"
  type        = number
  default     = 40
}

resource "huaweicloud_gaussdb_instance" "primary" {
  name                  = var.primary_instance_name
  flavor                = var.instance_flavor
  password              = try(random_password.instance[0].result, null)
  vpc_id                = huaweicloud_vpc.primary.id
  subnet_id             = try(huaweicloud_vpc_subnet.primary[0].id, null)
  security_group_id     = huaweicloud_networking_secgroup.primary.id
  availability_zone     = var.primary_instance_availability_zones
  port                  = var.instance_db_port
  enterprise_project_id = var.enterprise_project_id

  ha {
    mode             = "centralization_standard"
    replication_mode = "sync"
    consistency      = "strong"
  }

  replica_num = 3

  volume {
    type = var.primary_instance_volume_type
    size = var.primary_instance_volume_size
  }

  lifecycle {
    ignore_changes = [
      flavor,
    ]
  }
}
```

**Parameter Description**:
- **name**: The GaussDB instance name, assigned by referencing the input variable primary_instance_name
- **flavor**: The instance flavor, assigned by referencing the input variable instance_flavor
- **password**: The instance password, referencing the random password resource result
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing `huaweicloud_vpc.primary.id`
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing `huaweicloud_vpc_subnet.primary[0].id`
- **security_group_id**: The ID of the security group to which the instance belongs, referencing `huaweicloud_networking_secgroup.primary.id`
- **availability_zone**: The availability zone where the instance is located, assigned by referencing the input variable primary_instance_availability_zones
- **port**: The database port, assigned by referencing the input variable instance_db_port
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **ha.mode**: The high availability mode, which is `centralization_standard` here
- **ha.replication_mode**: The replication mode, which is `sync` here
- **ha.consistency**: The consistency mode, which is `strong` here
- **replica_num**: The number of replicas, which is `3` here
- **volume.type**: The storage type, assigned by referencing the input variable primary_instance_volume_type
- **volume.size**: The storage size, assigned by referencing the input variable primary_instance_volume_size

### 8. Create DR Region GaussDB Instance

Add the following script in the TF file (such as main.tf) to create the GaussDB instance in the DR region:

```hcl
# Create the GaussDB instance resource in the DR region in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dr_instance_name" {
  description = "The name of the DR GaussDB instance"
  type        = string
}

variable "dr_instance_availability_zones" {
  description = "The comma-separated AZ string for the DR GaussDB instance"
  type        = string
}

variable "dr_instance_volume_type" {
  description = "The storage volume type of the DR GaussDB instance"
  type        = string
  default     = "ULTRAHIGH"
}

variable "dr_instance_volume_size" {
  description = "The storage volume size (GB) of the DR GaussDB instance"
  type        = number
  default     = 40
}

resource "huaweicloud_gaussdb_instance" "dr" {
  provider = huaweicloud.dr

  name                  = var.dr_instance_name
  flavor                = var.instance_flavor
  password              = try(random_password.instance[1].result, null)
  vpc_id                = huaweicloud_vpc.dr.id
  subnet_id             = huaweicloud_vpc_subnet.dr.id
  security_group_id     = huaweicloud_networking_secgroup.dr.id
  availability_zone     = var.dr_instance_availability_zones
  port                  = var.instance_db_port
  enterprise_project_id = var.enterprise_project_id

  ha {
    mode             = "centralization_standard"
    replication_mode = "sync"
    consistency      = "strong"
  }

  replica_num = 3

  volume {
    type = var.dr_instance_volume_type
    size = var.dr_instance_volume_size
  }

  lifecycle {
    ignore_changes = [
      flavor,
    ]
  }
}
```

**Parameter Description**:
- **provider**: Specifies the provider with the alias `huaweicloud.dr` to create the instance in the DR region
- **name**: The GaussDB instance name, assigned by referencing the input variable dr_instance_name
- **flavor**: The instance flavor, assigned by referencing the input variable instance_flavor
- **password**: The instance password, referencing the random password resource result
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing `huaweicloud_vpc.dr.id`
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing `huaweicloud_vpc_subnet.dr.id`
- **security_group_id**: The ID of the security group to which the instance belongs, referencing `huaweicloud_networking_secgroup.dr.id`
- **availability_zone**: The availability zone where the instance is located, assigned by referencing the input variable dr_instance_availability_zones
- **port**: The database port, assigned by referencing the input variable instance_db_port
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **volume.type**: The storage type, assigned by referencing the input variable dr_instance_volume_type
- **volume.size**: The storage size, assigned by referencing the input variable dr_instance_volume_size

### 9. Reset DR Configuration

Add the following script in the TF file (such as main.tf) to reset the DR configuration of the primary and standby instances:

```hcl
# Reset the DR configuration of the primary instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_gaussdb_dr_configuration_reset" "primary" {
  instance_id        = huaweicloud_gaussdb_instance.primary.id
  opposite_data_cidr = huaweicloud_vpc_subnet.dr.cidr
}

resource "huaweicloud_gaussdb_dr_configuration_reset" "dr" {
  provider = huaweicloud.dr

  instance_id        = huaweicloud_gaussdb_instance.dr.id
  opposite_data_cidr = try(huaweicloud_vpc_subnet.primary[0].cidr, null)
}
```

**Parameter Description**:
- **instance_id**: The ID of the GaussDB instance whose DR configuration needs to be reset, referencing `huaweicloud_gaussdb_instance.primary.id` or `huaweicloud_gaussdb_instance.dr.id`
- **opposite_data_cidr**: The opposite data CIDR block; the primary instance references the DR region subnet CIDR, and the DR instance references the primary region subnet CIDR

### 10. Establish DR Relationship

Add the following script in the TF file (such as main.tf) to establish the DR relationship between the primary and standby instances:

```hcl
# Establish the GaussDB DR relationship in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dr_disaster_type" {
  description = "The disaster recovery type"
  type        = string
  default     = "stream"
}

variable "dr_user_name" {
  description = "The database username for disaster recovery"
  type        = string
}

variable "dr_user_password" {
  description = "The database password for disaster recovery"
  type        = string
  sensitive   = true
}

resource "huaweicloud_gaussdb_dr_relationship" "test" {
  instance_id      = huaweicloud_gaussdb_instance.primary.id
  disaster_type    = var.dr_disaster_type
  dr_ip            = try(huaweicloud_gaussdb_instance.dr.private_ips[0], null)
  dr_user_name     = var.dr_user_name
  dr_user_password = var.dr_user_password

  depends_on = [
    huaweicloud_gaussdb_dr_configuration_reset.primary,
    huaweicloud_gaussdb_dr_configuration_reset.dr,
    huaweicloud_cc_network_instance.primary,
    huaweicloud_cc_network_instance.dr,
    huaweicloud_cc_inter_region_bandwidth.dr,
  ]
}
```

**Parameter Description**:
- **instance_id**: The primary instance ID, referencing `huaweicloud_gaussdb_instance.primary.id`
- **disaster_type**: The disaster recovery type, assigned by referencing the input variable dr_disaster_type
- **dr_ip**: The private IP address of the DR instance, referencing `huaweicloud_gaussdb_instance.dr.private_ips[0]`
- **dr_user_name**: The DR database username, assigned by referencing the input variable dr_user_name
- **dr_user_password**: The DR database password, assigned by referencing the input variable dr_user_password
- **depends_on**: Explicitly declares dependencies to ensure the DR configuration reset and cloud connection network are ready before establishing the DR relationship

### 11. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Primary region network configuration
primary_vpc_name            = "example_vpc"
primary_vpc_cidr            = "172.16.0.0/16"
primary_subnet_names        = ["example_subnet_1", "example_subnet_2"]
primary_availability_zones  = ["primary_region_az_1", "primary_region_az_2"]
primary_security_group_name = "example_security_group"

# DR region network configuration
dr_vpc_name            = "example_vpc_dr"
dr_vpc_cidr            = "172.17.0.0/16"
dr_subnet_name         = "example_subnet_dr"
dr_availability_zone   = "dr_region_az"
dr_security_group_name = "example_security_group"
dr_region_name         = "dr_region"

# Cloud connection configuration
cc_bandwidth = 10

# Primary region GaussDB instance configuration
instance_passwords                  = ["password_1", "password_2"]
primary_instance_name               = "example_primary_instance"
instance_flavor                     = "gaussdb.opengauss.ee.c3.xlarge.x864.ha"
primary_instance_availability_zones = "primary_region_az_1,primary_region_az_2,primary_region_az_3"
instance_db_port                    = 5432
enterprise_project_id               = "your_enterprise_project_id"
primary_instance_volume_type        = "ULTRAHIGH"
primary_instance_volume_size        = 40

# DR region GaussDB instance configuration
dr_instance_name               = "example_dr_instance"
dr_instance_availability_zones = "dr_region_az_1,dr_region_az_2,dr_region_az_3"
dr_instance_volume_type        = "ULTRAHIGH"
dr_instance_volume_size        = 40

# DR relationship configuration
dr_user_name     = "root"
dr_user_password = "your_dr_user_password"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="primary_vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_primary_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the GaussDB disaster recovery architecture
4. Run `terraform show` to view the created GaussDB disaster recovery architecture

## Reference Information

- [Huawei Cloud GaussDB Product Documentation](https://support.huaweicloud.com/gaussdb/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GaussDB Disaster Recovery Architecture](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/dr-architecture)
