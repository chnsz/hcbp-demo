# Deploy Real-Time MySQL-to-DWS Data Synchronization

## Application Scenario

Data Warehouse Service (DWS) is an online analytical processing (OLAP) enterprise-level data warehouse service provided by Huawei Cloud, delivering high-performance, highly reliable, and easily scalable data warehouse capabilities for massive data analysis scenarios. In real-world business, source data is often stored in Relational Database Service (RDS) MySQL instances and needs to be synchronized to a DWS cluster in near real time for aggregation and analysis, supporting reporting, dashboards, and business intelligence (BI) scenarios.

This best practice will introduce how to use Terraform to automatically deploy the infrastructure for real-time MySQL-to-DWS data synchronization, including the creation of a shared VPC, subnet, and security group, an RDS MySQL instance, a DWS cluster, a DLI elastic resource pool and general queue, and a DLI enhanced datasource connection associated with the elastic resource pool, laying the foundation for implementing MySQL CDC to DWS real-time synchronization through a DLI Flink OpenSource SQL job.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS Flavors (data.huaweicloud_rds_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)
- [DWS Flavors (data.huaweicloud_dws_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dws_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RDS MySQL Instance (huaweicloud_rds_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DWS Cluster (huaweicloud_dws_cluster)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_cluster)
- [DLI Elastic Resource Pool (huaweicloud_dli_elastic_resource_pool)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_elastic_resource_pool)
- [DLI Queue (huaweicloud_dli_queue)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_queue)
- [DLI Enhanced Datasource Connection (huaweicloud_dli_datasource_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_datasource_connection)
- [DLI Enhanced Datasource Connection Associate (huaweicloud_dli_datasource_connection_associate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_datasource_connection_associate)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_rds_flavors
    │       └── huaweicloud_rds_instance
    └── data.huaweicloud_dws_flavors
            └── huaweicloud_dws_cluster

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            ├── huaweicloud_rds_instance
            ├── huaweicloud_dws_cluster
            └── huaweicloud_dli_datasource_connection

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    ├── huaweicloud_rds_instance
    └── huaweicloud_dws_cluster

huaweicloud_dli_elastic_resource_pool
    ├── huaweicloud_dli_queue
    └── huaweicloud_dli_datasource_connection_associate

huaweicloud_dli_datasource_connection
    └── huaweicloud_dli_datasource_connection_associate
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the article [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query the Availability Zone List

Add the following script to the TF file (such as main.tf) to query the availability zone list in the current region, which is used to specify the availability zone when creating the RDS instance and DWS cluster:

```hcl
# Query the availability zone list in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zone" {
  description = "The availability zone. If empty, the first available zone is used"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**Parameter Description**:

- **count**: The data source is created when the input variable `availability_zone` is empty, to automatically obtain the availability zone list in the current region

### 3. Create a Virtual Private Cloud

Add the following script to the TF file to create the virtual private cloud shared by RDS and DWS:

```hcl
# Create a virtual private cloud in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC shared by RDS and DWS"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC. Must differ from the DLI elastic resource pool CIDR"
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

- **name**: Assigned by referencing the input variable `vpc_name`, used to specify the name of the virtual private cloud
- **cidr**: Assigned by referencing the input variable `vpc_cidr`, used to specify the CIDR block of the virtual private cloud, which must differ from the DLI elastic resource pool CIDR
- **enterprise_project_id**: Assigned by referencing the input variable `enterprise_project_id`, used to specify the enterprise project ID

### 4. Create a VPC Subnet

Add the following script to the TF file to create a subnet:

```hcl
# Create a VPC subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet. If empty, it is calculated from the VPC CIDR"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet. If empty, it is calculated from the subnet CIDR"
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

- **vpc_id**: Assigned by referencing `huaweicloud_vpc.test.id`, used to specify the virtual private cloud to which the subnet belongs
- **name**: Assigned by referencing the input variable `subnet_name`, used to specify the name of the subnet
- **cidr**: Assigned by referencing the input variable `subnet_cidr`; if empty, it is calculated from the VPC CIDR
- **gateway_ip**: Assigned by referencing the input variable `subnet_gateway_ip`; if empty, it is calculated from the subnet CIDR

### 5. Create a Security Group and Its Rule

Add the following script to the TF file to create a security group and allow the DLI elastic resource pool CIDR to access the DWS and RDS ports:

```hcl
# Create a security group and its rule in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group shared by the RDS instance and DWS cluster"
  type        = string
}

variable "security_group_delete_default_rules" {
  description = "Whether to delete the default rules of the security group"
  type        = bool
  default     = true
}

variable "dws_port" {
  description = "The service port of the DWS cluster"
  type        = number
  default     = 8000
}

variable "rds_db_port" {
  description = "The database port of the RDS MySQL instance"
  type        = number
  default     = 3306
}

variable "elastic_resource_pool_cidr" {
  description = "The CIDR block of the DLI elastic resource pool. Must differ from the VPC CIDR"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                  = var.security_group_name
  delete_default_rules  = var.security_group_delete_default_rules
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}

# DWS and RDS ports are opened to DLI elastic resource pool
resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  ports             = "${var.dws_port},${var.rds_db_port}"
  remote_ip_prefix  = var.elastic_resource_pool_cidr
}
```

**Parameter Description**:

- **name**: Assigned by referencing the input variable `security_group_name`, used to specify the name of the security group
- **delete_default_rules**: Assigned by referencing the input variable `security_group_delete_default_rules`, used to specify whether to delete the default rules of the security group
- **security_group_id**: Assigned by referencing `huaweicloud_networking_secgroup.test.id`, used to specify the security group to which the rule belongs
- **direction**: Set to `ingress`, indicating an inbound rule
- **protocol**: Set to `tcp`, indicating the TCP protocol
- **ports**: Assigned by referencing the input variables `dws_port` and `rds_db_port`, used to open the DWS and RDS service ports
- **remote_ip_prefix**: Assigned by referencing the input variable `elastic_resource_pool_cidr`, used to specify the DLI elastic resource pool CIDR

### 6. Query the RDS Flavor List

Add the following script to the TF file to query RDS MySQL instance flavors:

```hcl
# Query the RDS flavor list in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "rds_flavor_id" {
  description = "The flavor ID of the RDS instance. If empty, it is queried from huaweicloud_rds_flavors"
  type        = string
  default     = ""
  nullable    = false
}

variable "rds_db_version" {
  description = "The MySQL version of the RDS instance"
  type        = string
  default     = "5.7"
}

variable "rds_instance_mode" {
  description = "The instance mode used to query RDS flavors"
  type        = string
  default     = "single"
}

variable "rds_flavor_vcpus" {
  description = "The vCPUs used to query RDS flavors"
  type        = number
  default     = 2
}

data "huaweicloud_rds_flavors" "test" {
  count = var.rds_flavor_id == "" ? 1 : 0

  db_type           = "MySQL"
  db_version        = var.rds_db_version
  instance_mode     = var.rds_instance_mode
  vcpus             = var.rds_flavor_vcpus
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**Parameter Description**:

- **count**: The data source is created when the input variable `rds_flavor_id` is empty, to automatically query RDS flavors
- **db_type**: Set to `MySQL`, indicating that MySQL flavors are queried
- **db_version**: Assigned by referencing the input variable `rds_db_version`, used to specify the MySQL version
- **instance_mode**: Assigned by referencing the input variable `rds_instance_mode`, used to specify the instance mode
- **vcpus**: Assigned by referencing the input variable `rds_flavor_vcpus`, used to specify the number of vCPUs of the flavor
- **availability_zone**: Assigned by referencing the input variable `availability_zone` or the availability zone list data source

### 7. Create an RDS MySQL Instance

Add the following script to the TF file to create an RDS MySQL instance:

```hcl
# Create an RDS MySQL instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "rds_instance_name" {
  description = "The name of the RDS MySQL instance"
  type        = string
}

variable "rds_db_password" {
  description = "The root password of the RDS MySQL instance"
  type        = string
  sensitive   = true
}

variable "rds_volume_type" {
  description = "The volume type of the RDS instance"
  type        = string
  default     = "CLOUDSSD"
}

variable "rds_volume_size" {
  description = "The volume size of the RDS instance in GB"
  type        = number
  default     = 40
}

resource "huaweicloud_rds_instance" "test" {
  name              = var.rds_instance_name
  flavor            = var.rds_flavor_id != "" ? var.rds_flavor_id : try(data.huaweicloud_rds_flavors.test[0].flavors[0].name, null)
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  availability_zone = var.availability_zone != "" ? [var.availability_zone] : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1), null)

  db {
    type     = "MySQL"
    version  = var.rds_db_version
    port     = var.rds_db_port
    password = var.rds_db_password
  }

  volume {
    type = var.rds_volume_type
    size = var.rds_volume_size
  }

  lifecycle {
    ignore_changes = [flavor]
  }
}
```

**Parameter Description**:

- **name**: Assigned by referencing the input variable `rds_instance_name`, used to specify the name of the RDS instance
- **flavor**: Assigned by referencing the input variable `rds_flavor_id` or the RDS flavor list data source
- **vpc_id**: Assigned by referencing `huaweicloud_vpc.test.id`, used to specify the virtual private cloud to which the instance belongs
- **subnet_id**: Assigned by referencing `huaweicloud_vpc_subnet.test.id`, used to specify the subnet to which the instance belongs
- **security_group_id**: Assigned by referencing `huaweicloud_networking_secgroup.test.id`, used to specify the security group to which the instance belongs
- **availability_zone**: Assigned by referencing the input variable `availability_zone` or the availability zone list data source
- **db.type**: Set to `MySQL`, indicating the database type
- **db.version**: Assigned by referencing the input variable `rds_db_version`, used to specify the database version
- **db.port**: Assigned by referencing the input variable `rds_db_port`, used to specify the database port
- **db.password**: Assigned by referencing the input variable `rds_db_password`, used to specify the root password of the database
- **volume.type**: Assigned by referencing the input variable `rds_volume_type`, used to specify the storage type
- **volume.size**: Assigned by referencing the input variable `rds_volume_size`, used to specify the storage size

### 8. Query the DWS Flavor List

Add the following script to the TF file to query DWS cluster flavors:

```hcl
# Query the DWS flavor list in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dws_node_type" {
  description = "The flavor of the DWS cluster node. If empty, it is queried from huaweicloud_dws_flavors"
  type        = string
  default     = ""
  nullable    = false
}

variable "dws_version" {
  description = "The version of the DWS cluster. If empty, it is queried from huaweicloud_dws_flavors"
  type        = string
  default     = ""
  nullable    = false
}

variable "dws_flavor_vcpus" {
  description = "The vCPUs used to query DWS flavors"
  type        = number
  default     = 4
}

variable "dws_flavor_memory" {
  description = "The memory used to query DWS flavors"
  type        = number
  default     = 32
}

variable "dws_datastore_type" {
  description = "The datastore type of the DWS cluster"
  type        = string
  default     = "dws"
}

data "huaweicloud_dws_flavors" "test" {
  count = var.dws_node_type == "" || var.dws_version == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  vcpus             = var.dws_flavor_vcpus
  memory            = var.dws_flavor_memory
  datastore_type    = var.dws_datastore_type
}
```

**Parameter Description**:

- **count**: The data source is created when the input variable `dws_node_type` or `dws_version` is empty, to automatically query DWS flavors
- **availability_zone**: Assigned by referencing the input variable `availability_zone` or the availability zone list data source
- **vcpus**: Assigned by referencing the input variable `dws_flavor_vcpus`, used to specify the number of vCPUs of the flavor
- **memory**: Assigned by referencing the input variable `dws_flavor_memory`, used to specify the memory size of the flavor
- **datastore_type**: Assigned by referencing the input variable `dws_datastore_type`, used to specify the datastore type

### 9. Create a DWS Cluster

Add the following script to the TF file to create a DWS cluster:

```hcl
# Create a DWS cluster in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dws_cluster_name" {
  description = "The name of the DWS cluster"
  type        = string
}

variable "dws_number_of_node" {
  description = "The number of nodes in the DWS cluster"
  type        = number
  default     = 3
}

variable "dws_number_of_cn" {
  description = "The number of CN nodes in the DWS cluster"
  type        = number
  default     = 3
}

variable "dws_admin_user_name" {
  description = "The administrator username of the DWS cluster"
  type        = string
  default     = "dbadmin"
}

variable "dws_admin_user_pwd" {
  description = "The administrator password of the DWS cluster"
  type        = string
  sensitive   = true
}

variable "dws_volume_type" {
  description = "The volume type of the DWS cluster"
  type        = string
  default     = "SSD"
}

variable "dws_volume_capacity" {
  description = "The volume capacity of the DWS cluster in GB"
  type        = string
  default     = "100"
}

resource "huaweicloud_dws_cluster" "test" {
  name                  = var.dws_cluster_name
  node_type             = var.dws_node_type != "" ? var.dws_node_type : try(data.huaweicloud_dws_flavors.test[0].flavors[0].flavor_id, null)
  number_of_node        = var.dws_number_of_node
  number_of_cn          = var.dws_number_of_cn
  version               = var.dws_version != "" ? var.dws_version : try(data.huaweicloud_dws_flavors.test[0].flavors[0].datastore_version, null)
  vpc_id                = huaweicloud_vpc.test.id
  network_id            = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  user_name             = var.dws_admin_user_name
  user_pwd              = var.dws_admin_user_pwd
  port                  = var.dws_port
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  volume {
    type     = var.dws_volume_type
    capacity = var.dws_volume_capacity
  }
}
```

**Parameter Description**:

- **name**: Assigned by referencing the input variable `dws_cluster_name`, used to specify the name of the DWS cluster
- **node_type**: Assigned by referencing the input variable `dws_node_type` or the DWS flavor list data source
- **number_of_node**: Assigned by referencing the input variable `dws_number_of_node`, used to specify the number of cluster nodes
- **number_of_cn**: Assigned by referencing the input variable `dws_number_of_cn`, used to specify the number of CN nodes
- **version**: Assigned by referencing the input variable `dws_version` or the DWS flavor list data source
- **vpc_id**: Assigned by referencing `huaweicloud_vpc.test.id`, used to specify the virtual private cloud to which the cluster belongs
- **network_id**: Assigned by referencing `huaweicloud_vpc_subnet.test.id`, used to specify the subnet to which the cluster belongs
- **security_group_id**: Assigned by referencing `huaweicloud_networking_secgroup.test.id`, used to specify the security group to which the cluster belongs
- **availability_zone**: Assigned by referencing the input variable `availability_zone` or the availability zone list data source
- **user_name**: Assigned by referencing the input variable `dws_admin_user_name`, used to specify the cluster administrator username
- **user_pwd**: Assigned by referencing the input variable `dws_admin_user_pwd`, used to specify the cluster administrator password
- **port**: Assigned by referencing the input variable `dws_port`, used to specify the cluster service port
- **volume.type**: Assigned by referencing the input variable `dws_volume_type`, used to specify the storage type
- **volume.capacity**: Assigned by referencing the input variable `dws_volume_capacity`, used to specify the storage capacity

### 10. Create a DLI Elastic Resource Pool

Add the following script to the TF file to create a DLI elastic resource pool:

```hcl
# Create a DLI elastic resource pool in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "elastic_resource_pool_name" {
  description = "The name of the DLI elastic resource pool"
  type        = string
}

variable "elastic_resource_pool_description" {
  description = "The description of the DLI elastic resource pool"
  type        = string
  default     = ""
}

variable "elastic_resource_pool_min_cu" {
  description = "The minimum number of CUs for the DLI elastic resource pool"
  type        = number
  default     = 16
}

variable "elastic_resource_pool_max_cu" {
  description = "The maximum number of CUs for the DLI elastic resource pool"
  type        = number
  default     = 64
}

variable "elastic_resource_pool_label" {
  description = "The label of the DLI elastic resource pool"
  type        = map(string)

  default = {
    spec = "basic"
  }
}

resource "huaweicloud_dli_elastic_resource_pool" "test" {
  name                  = var.elastic_resource_pool_name
  description           = var.elastic_resource_pool_description
  min_cu                = var.elastic_resource_pool_min_cu
  max_cu                = var.elastic_resource_pool_max_cu
  cidr                  = var.elastic_resource_pool_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  label                 = var.elastic_resource_pool_label
}
```

**Parameter Description**:

- **name**: Assigned by referencing the input variable `elastic_resource_pool_name`, used to specify the name of the elastic resource pool
- **description**: Assigned by referencing the input variable `elastic_resource_pool_description`, used to specify the description of the elastic resource pool
- **min_cu**: Assigned by referencing the input variable `elastic_resource_pool_min_cu`, used to specify the minimum number of CUs of the elastic resource pool
- **max_cu**: Assigned by referencing the input variable `elastic_resource_pool_max_cu`, used to specify the maximum number of CUs of the elastic resource pool
- **cidr**: Assigned by referencing the input variable `elastic_resource_pool_cidr`, used to specify the CIDR block of the elastic resource pool, which must differ from the VPC CIDR
- **label**: Assigned by referencing the input variable `elastic_resource_pool_label`, used to specify the label of the elastic resource pool

### 11. Create a DLI Queue

Add the following script to the TF file to create a DLI general queue:

```hcl
# Create a DLI queue in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "queue_name" {
  description = "The name of the DLI general queue"
  type        = string
}

variable "queue_cu_count" {
  description = "The CU count of the DLI queue"
  type        = number
  default     = 16
}

variable "queue_description" {
  description = "The description of the DLI queue"
  type        = string
  default     = ""
}

resource "huaweicloud_dli_queue" "test" {
  elastic_resource_pool_name = huaweicloud_dli_elastic_resource_pool.test.name
  resource_mode              = 1
  name                       = var.queue_name
  queue_type                 = "general"
  cu_count                   = var.queue_cu_count
  description                = var.queue_description
  enterprise_project_id      = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:

- **elastic_resource_pool_name**: Assigned by referencing `huaweicloud_dli_elastic_resource_pool.test.name`, used to specify the elastic resource pool to which the queue belongs
- **resource_mode**: Set to `1`, indicating that the queue uses the elastic resource pool mode
- **name**: Assigned by referencing the input variable `queue_name`, used to specify the name of the queue
- **queue_type**: Set to `general`, indicating a general queue
- **cu_count**: Assigned by referencing the input variable `queue_cu_count`, used to specify the CU count of the queue
- **description**: Assigned by referencing the input variable `queue_description`, used to specify the description of the queue

### 12. Create a DLI Enhanced Datasource Connection and Associate It with the Elastic Resource Pool

Add the following script to the TF file to create a DLI enhanced datasource connection and associate it with the elastic resource pool:

```hcl
# Create a DLI enhanced datasource connection and associate it with the elastic resource pool in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "datasource_connection_name" {
  description = "The name of the DLI enhanced datasource connection"
  type        = string
}

resource "huaweicloud_dli_datasource_connection" "test" {
  name      = var.datasource_connection_name
  vpc_id    = huaweicloud_vpc.test.id
  subnet_id = huaweicloud_vpc_subnet.test.id
}

resource "huaweicloud_dli_datasource_connection_associate" "test" {
  connection_id          = huaweicloud_dli_datasource_connection.test.id
  elastic_resource_pools = [huaweicloud_dli_elastic_resource_pool.test.name]

  depends_on = [huaweicloud_dli_queue.test]
}
```

**Parameter Description**:

- **name**: Assigned by referencing the input variable `datasource_connection_name`, used to specify the name of the enhanced datasource connection
- **vpc_id**: Assigned by referencing `huaweicloud_vpc.test.id`, used to specify the virtual private cloud to which the datasource connection belongs
- **subnet_id**: Assigned by referencing `huaweicloud_vpc_subnet.test.id`, used to specify the subnet to which the datasource connection belongs
- **connection_id**: Assigned by referencing `huaweicloud_dli_datasource_connection.test.id`, used to specify the datasource connection to be associated
- **elastic_resource_pools**: Assigned by referencing `huaweicloud_dli_elastic_resource_pool.test.name`, used to specify the elastic resource pool to be associated
- **depends_on**: Explicitly declares the dependency to ensure the association is performed after the queue is created

### 13. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name                   = "tf_test_mysql_dws_vpc"
vpc_cidr                   = "192.168.0.0/16"
subnet_name                = "tf_test_mysql_dws_subnet"
security_group_name        = "tf_test_mysql_dws_sg"
elastic_resource_pool_cidr = "172.16.0.0/18"
rds_instance_name          = "tf-test-rds-mysql"
rds_db_password            = "YourPassword@123"
dws_cluster_name           = "tf-test-dws-cluster"
dws_admin_user_pwd         = "YourPassword@123"
elastic_resource_pool_name = "tf_test_dli_pool"
queue_name                 = "tf_test_dli_queue"
datasource_connection_name = "tf_test_dli_conn"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows the user to automatically import the content of the `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 14. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the infrastructure required for real-time MySQL-to-DWS data synchronization
4. Run `terraform show` to view the created resources

> Note: After the infrastructure is created, you also need to create the source table in RDS MySQL, create the target table in DWS, prepare the DWS Connector JAR and upload it to OBS, and then create a Flink OpenSource SQL job in DLI using `mysql-cdc` as the source connector and `gaussdb` as the sink connector to implement real-time MySQL-to-DWS data synchronization.

## Reference Information

- [Huawei Cloud Data Warehouse Service Product Documentation](https://support.huaweicloud.com/dws/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DWS Real-Time MySQL-to-DWS Data Synchronization](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dws/real-time-sync-mysql-to-dws)
