# Deploy Workload Management

## Application Scenario

Data Warehouse Service (DWS) is an online analytical processing (OLAP) enterprise-level data warehouse service provided by Huawei Cloud, delivering high-performance, highly reliable, and easily scalable data warehouse capabilities for massive data analysis scenarios. When multiple businesses or tenants share the same DWS cluster, workload management is required to isolate and schedule resources such as CPU, memory, and concurrency, preventing individual businesses from monopolizing resources and affecting overall stability.

This best practice will introduce how to use Terraform to automatically deploy DWS workload management, including VPC, subnet, and security group creation, DWS cluster creation, workload queue configuration, cluster user creation, queue-user association, workload plan and plan stage configuration, and exception rule configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DWS Cluster Flavors (data.huaweicloud_dws_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dws_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [DWS Cluster (huaweicloud_dws_cluster)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_cluster)
- [DWS Workload Queue (huaweicloud_dws_workload_queue)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_workload_queue)
- [DWS Cluster User (huaweicloud_dws_cluster_user)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_cluster_user)
- [DWS Workload Queue User Associate (huaweicloud_dws_workload_queue_user_associate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_workload_queue_user_associate)
- [DWS Workload Plan (huaweicloud_dws_workload_plan)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_workload_plan)
- [DWS Workload Plan Stage (huaweicloud_dws_workload_plan_stage)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_workload_plan_stage)
- [DWS Cluster Exception Rule (huaweicloud_dws_cluster_exception_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_cluster_exception_rule)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dws_cluster

data.huaweicloud_dws_flavors
    └── huaweicloud_dws_cluster

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dws_cluster

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
    └── huaweicloud_dws_cluster

huaweicloud_dws_cluster
    └── huaweicloud_dws_workload_queue
    └── huaweicloud_dws_cluster_user
    └── huaweicloud_dws_workload_plan
    └── huaweicloud_dws_cluster_exception_rule

huaweicloud_dws_workload_queue
    └── huaweicloud_dws_workload_queue_user_associate
    └── huaweicloud_dws_workload_plan_stage

huaweicloud_dws_cluster_user
    └── huaweicloud_dws_workload_queue_user_associate

huaweicloud_dws_workload_plan
    └── huaweicloud_dws_workload_plan_stage
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones available for the DWS cluster:

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter description**:
- **count**: Determines whether to query the availability zones by referencing the input variable availability_zone when no availability zone is specified

### 3. Create a VPC

Add the following script in the TF file (such as main.tf) to create a VPC:

```hcl
# Create a VPC in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter description**:
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; no enterprise project is specified when it is empty

### 4. Create a VPC Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create a VPC subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter description**:
- **vpc_id**: Assigned by referencing the ID of the resource huaweicloud_vpc.test
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; the subnet is automatically divided based on the VPC CIDR when it is empty
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

**Parameter description**:
- **name**: Assigned by referencing the input variable security_group_name
- **delete_default_rules**: Assigned by referencing the input variable security_group_delete_default_rules
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; no enterprise project is specified when it is empty

### 6. Create a Security Group Rule

Add the following script in the TF file (such as main.tf) to create a security group rule that allows DWS clients to access the cluster:

```hcl
# Create a security group rule in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_rule_ports" {
  description = "The security group ingress rule ports for DWS"
  type        = string
  default     = "8000-10000"
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = huaweicloud_vpc.test.cidr
  ports             = var.security_group_rule_ports
  protocol          = "tcp"
}
```

**Parameter description**:
- **security_group_id**: Assigned by referencing the ID of the resource huaweicloud_networking_secgroup.test
- **direction**: The direction of the security group rule, which is ingress
- **ethertype**: The network type, which is IPv4
- **remote_ip_prefix**: Assigned by referencing the CIDR of the resource huaweicloud_vpc.test
- **ports**: Assigned by referencing the input variable security_group_rule_ports
- **protocol**: The protocol type, which is tcp

### 7. Query DWS Cluster Flavors

Add the following script in the TF file (such as main.tf) to query the DWS cluster flavors:

```hcl
# Query the DWS cluster flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter description**:
- **count**: Determines whether to query the cluster flavors by referencing the input variables cluster_node_type and cluster_version when no node flavor or cluster version is specified
- **availability_zone**: Assigned by referencing the input variable availability_zone or the availability zone of the data source huaweicloud_availability_zones.test
- **vcpus**: Assigned by referencing the input variable cluster_vcpus
- **memory**: Assigned by referencing the input variable cluster_memory
- **datastore_type**: Assigned by referencing the input variable cluster_datastore_type

### 8. Create a DWS Cluster

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

**Parameter description**:
- **name**: Assigned by referencing the input variable cluster_name
- **node_type**: Assigned by referencing the input variable cluster_node_type or the flavor ID of the data source huaweicloud_dws_flavors.test
- **number_of_node**: Assigned by referencing the input variable cluster_number_of_node
- **number_of_cn**: Assigned by referencing the input variable cluster_number_of_cn
- **version**: Assigned by referencing the input variable cluster_version or the version of the data source huaweicloud_dws_flavors.test
- **vpc_id**: Assigned by referencing the ID of the resource huaweicloud_vpc.test
- **network_id**: Assigned by referencing the ID of the resource huaweicloud_vpc_subnet.test
- **security_group_id**: Assigned by referencing the ID of the resource huaweicloud_networking_secgroup.test
- **availability_zone**: Assigned by referencing the input variable availability_zone or the availability zone of the data source huaweicloud_availability_zones.test
- **user_name**: Assigned by referencing the input variable cluster_admin_user_name
- **user_pwd**: Assigned by referencing the input variable cluster_admin_user_pwd
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id; no enterprise project is specified when it is empty
- **volume.type**: Assigned by referencing the input variable cluster_volume_type
- **volume.capacity**: Assigned by referencing the input variable cluster_volume_capacity

### 9. Create a Workload Queue

Add the following script in the TF file (such as main.tf) to create a workload queue:

```hcl
# Create a workload queue in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "workload_queue_name" {
  description = "The name of the workload queue"
  type        = string
}

variable "workload_queue_configurations" {
  description = "The configurations of the workload queue"

  type = list(object({
    resource_name  = string
    resource_value = string
  }))
}

resource "huaweicloud_dws_workload_queue" "test" {
  cluster_id = huaweicloud_dws_cluster.test.id
  name       = var.workload_queue_name

  dynamic "configuration" {
    for_each = var.workload_queue_configurations

    content {
      resource_name  = configuration.value.resource_name
      resource_value = configuration.value.resource_value
    }
  }
}
```

**Parameter description**:
- **cluster_id**: Assigned by referencing the ID of the resource huaweicloud_dws_cluster.test
- **name**: Assigned by referencing the input variable workload_queue_name
- **configuration.resource_name**: Assigned by referencing the resource_name in the input variable workload_queue_configurations
- **configuration.resource_value**: Assigned by referencing the resource_value in the input variable workload_queue_configurations

### 10. Create a Cluster User

Add the following script in the TF file (such as main.tf) to create a cluster user:

```hcl
# Create a cluster user in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "user_name" {
  description = "The name of the cluster user"
  type        = string
}

variable "user_password" {
  description = "The password of the cluster user"
  type        = string
  sensitive   = true
}

variable "user_description" {
  description = "The description of the cluster user"
  type        = string
  default     = ""
  nullable    = false
}

variable "user_cascade" {
  description = "Whether to cascade delete dependencies when deleting the user or role"
  type        = bool
  default     = true
}

variable "user_login" {
  description = "Whether to allow the user to log in"
  type        = bool
  default     = true
}

variable "user_create_role" {
  description = "Whether to grant the permission to create roles"
  type        = bool
  default     = true
}

variable "user_create_db" {
  description = "Whether to grant the permission to create databases"
  type        = bool
  default     = true
}

variable "user_system_admin" {
  description = "Whether to grant the system administrator permission"
  type        = bool
  default     = null
}

variable "user_audit_admin" {
  description = "Whether to grant the audit administrator permission"
  type        = bool
  default     = null
}

variable "user_inherit" {
  description = "Whether to inherit permissions from roles"
  type        = bool
  default     = true
}

variable "user_use_ft" {
  description = "Whether to grant the external table permission"
  type        = bool
  default     = null
}

variable "user_conn_limit" {
  description = "The maximum number of concurrent connections. -1 means unlimited"
  type        = number
  default     = -1
}

variable "user_replication" {
  description = "Whether to grant the replication permission"
  type        = bool
  default     = null
}

variable "user_valid_begin" {
  description = "The valid begin time of the cluster user"
  type        = string
  default     = ""
  nullable    = false
}

variable "user_valid_until" {
  description = "The valid until time of the cluster user"
  type        = string
  default     = ""
  nullable    = false
}

variable "user_grant_list" {
  description = "The set of grants for the cluster user"

  type = list(object({
    type                 = string
    database             = optional(string)
    schema_name          = optional(string)
    object_name          = optional(string)
    all_object           = optional(bool)
    future               = optional(bool)
    future_object_owners = optional(string)
    column_names         = optional(list(string))

    privileges = list(object({
      permission = string
      grant_with = bool
    }))
  }))

  default = []
}

resource "huaweicloud_dws_cluster_user" "test" {
  cluster_id   = huaweicloud_dws_cluster.test.id
  type         = "user"
  name         = var.user_name
  password     = var.user_password
  description  = var.user_description != "" ? var.user_description : null
  cascade      = var.user_cascade
  login        = var.user_login
  create_role  = var.user_create_role
  create_db    = var.user_create_db
  system_admin = var.user_system_admin
  audit_admin  = var.user_audit_admin
  inherit      = var.user_inherit
  use_ft       = var.user_use_ft
  conn_limit   = var.user_conn_limit
  replication  = var.user_replication
  valid_begin  = var.user_valid_begin
  valid_until  = var.user_valid_until

  dynamic "grant_list" {
    for_each = var.user_grant_list

    content {
      type                 = grant_list.value.type
      database             = grant_list.value.database
      schema_name          = grant_list.value.schema_name
      object_name          = grant_list.value.object_name
      all_object           = grant_list.value.all_object
      future               = grant_list.value.future
      future_object_owners = grant_list.value.future_object_owners
      column_names         = grant_list.value.column_names

      dynamic "privileges" {
        for_each = grant_list.value.privileges

        content {
          permission = privileges.value.permission
          grant_with = privileges.value.grant_with
        }
      }
    }
  }
}
```

**Parameter description**:
- **cluster_id**: Assigned by referencing the ID of the resource huaweicloud_dws_cluster.test
- **type**: The user type, which is user
- **name**: Assigned by referencing the input variable user_name
- **password**: Assigned by referencing the input variable user_password
- **description**: Assigned by referencing the input variable user_description; no description is specified when it is empty
- **cascade**: Assigned by referencing the input variable user_cascade
- **login**: Assigned by referencing the input variable user_login
- **create_role**: Assigned by referencing the input variable user_create_role
- **create_db**: Assigned by referencing the input variable user_create_db
- **system_admin**: Assigned by referencing the input variable user_system_admin
- **audit_admin**: Assigned by referencing the input variable user_audit_admin
- **inherit**: Assigned by referencing the input variable user_inherit
- **use_ft**: Assigned by referencing the input variable user_use_ft
- **conn_limit**: Assigned by referencing the input variable user_conn_limit
- **replication**: Assigned by referencing the input variable user_replication
- **valid_begin**: Assigned by referencing the input variable user_valid_begin
- **valid_until**: Assigned by referencing the input variable user_valid_until
- **grant_list**: Assigned by referencing the input variable user_grant_list to configure the grants of the user

### 11. Associate the Workload Queue with the User

Add the following script in the TF file (such as main.tf) to associate the workload queue with the user:

```hcl
# Associate the workload queue with the user in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_dws_workload_queue_user_associate" "test" {
  cluster_id = huaweicloud_dws_cluster.test.id
  queue_name = huaweicloud_dws_workload_queue.test.name
  user_names = [huaweicloud_dws_cluster_user.test.name]
}
```

**Parameter description**:
- **cluster_id**: Assigned by referencing the ID of the resource huaweicloud_dws_cluster.test
- **queue_name**: Assigned by referencing the name of the resource huaweicloud_dws_workload_queue.test
- **user_names**: Assigned by referencing the name of the resource huaweicloud_dws_cluster_user.test

### 12. Create a Workload Plan

Add the following script in the TF file (such as main.tf) to create a workload plan:

```hcl
# Create a workload plan in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "workload_plan_name" {
  description = "The name of the workload plan"
  type        = string
}

resource "huaweicloud_dws_workload_plan" "test" {
  cluster_id = huaweicloud_dws_cluster.test.id
  name       = var.workload_plan_name
}
```

**Parameter description**:
- **cluster_id**: Assigned by referencing the ID of the resource huaweicloud_dws_cluster.test
- **name**: Assigned by referencing the input variable workload_plan_name

### 13. Create a Workload Plan Stage

Add the following script in the TF file (such as main.tf) to create a workload plan stage:

```hcl
# Create a workload plan stage in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "workload_plan_stage_name" {
  description = "The name of the workload plan stage"
  type        = string
}

variable "workload_plan_stage_month" {
  description = "The month of the workload plan stage"
  type        = string
  default     = null
}

variable "workload_plan_stage_day" {
  description = "The day of the workload plan stage"
  type        = string
  default     = null
}

variable "workload_plan_stage_start_time" {
  description = "The start time of the workload plan stage. The format is hh:mm:ss"
  type        = string
  default     = "00:00:00"
}

variable "workload_plan_stage_end_time" {
  description = "The end time of the workload plan stage. The format is hh:mm:ss"
  type        = string
  default     = "23:59:59"
}

variable "workload_plan_stage_configurations" {
  description = "The configurations of the workload plan stage"

  type = list(object({
    resource_name        = string
    resource_value       = string
    value_unit           = optional(string)
    resource_description = optional(string)
  }))
}

resource "huaweicloud_dws_workload_plan_stage" "test" {
  cluster_id = huaweicloud_dws_cluster.test.id
  plan_id    = huaweicloud_dws_workload_plan.test.id
  name       = var.workload_plan_stage_name
  month      = var.workload_plan_stage_month
  day        = var.workload_plan_stage_day
  start_time = var.workload_plan_stage_start_time
  end_time   = var.workload_plan_stage_end_time

  queues {
    name = huaweicloud_dws_workload_queue.test.name

    dynamic "configuration" {
      for_each = var.workload_plan_stage_configurations

      content {
        resource_name        = configuration.value.resource_name
        resource_value       = configuration.value.resource_value
        value_unit           = configuration.value.value_unit
        resource_description = configuration.value.resource_description
      }
    }
  }
}
```

**Parameter description**:
- **cluster_id**: Assigned by referencing the ID of the resource huaweicloud_dws_cluster.test
- **plan_id**: Assigned by referencing the ID of the resource huaweicloud_dws_workload_plan.test
- **name**: Assigned by referencing the input variable workload_plan_stage_name
- **month**: Assigned by referencing the input variable workload_plan_stage_month; it applies to all months when not specified
- **day**: Assigned by referencing the input variable workload_plan_stage_day; it applies to all days when not specified
- **start_time**: Assigned by referencing the input variable workload_plan_stage_start_time
- **end_time**: Assigned by referencing the input variable workload_plan_stage_end_time
- **queues.name**: Assigned by referencing the name of the resource huaweicloud_dws_workload_queue.test
- **configuration.resource_name**: Assigned by referencing the resource_name in the input variable workload_plan_stage_configurations
- **configuration.resource_value**: Assigned by referencing the resource_value in the input variable workload_plan_stage_configurations
- **configuration.value_unit**: Assigned by referencing the value_unit in the input variable workload_plan_stage_configurations
- **configuration.resource_description**: Assigned by referencing the resource_description in the input variable workload_plan_stage_configurations

### 14. Create a Cluster Exception Rule

Add the following script in the TF file (such as main.tf) to create a cluster exception rule:

```hcl
# Create a cluster exception rule in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "exception_rule_name" {
  description = "The name of the cluster exception rule"
  type        = string
}

variable "exception_rule_configurations" {
  description = "The configurations of the exception rule"

  type = list(object({
    key   = string
    value = string
  }))
}

resource "huaweicloud_dws_cluster_exception_rule" "test" {
  cluster_id = huaweicloud_dws_cluster.test.id
  name       = var.exception_rule_name

  dynamic "configurations" {
    for_each = var.exception_rule_configurations

    content {
      key   = configurations.value.key
      value = configurations.value.value
    }
  }
}
```

**Parameter description**:
- **cluster_id**: Assigned by referencing the ID of the resource huaweicloud_dws_cluster.test
- **name**: Assigned by referencing the input variable exception_rule_name
- **configurations.key**: Assigned by referencing the key in the input variable exception_rule_configurations
- **configurations.value**: Assigned by referencing the value in the input variable exception_rule_configurations

### 15. Preset Input Parameters Required for Resource Deployment (Optional)

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
cluster_name            = "tf_test_cluster"
cluster_admin_user_name = "dbadmin"
cluster_admin_user_pwd  = "YourPassword@123"
workload_queue_name     = "tf_test_queue"

workload_queue_configurations = [
  {
    resource_name  = "cpu_limit"
    resource_value = "10"
  },
  {
    resource_name  = "memory"
    resource_value = "10"
  },
  {
    resource_name  = "tablespace"
    resource_value = "-1"
  },
  {
    resource_name  = "activestatements"
    resource_value = "-1"
  }
]

user_name                = "tf_test_user"
user_password            = "YourPassword@456"
workload_plan_name       = "tf_test_plan"
workload_plan_stage_name = "tf_test_stage"

workload_plan_stage_configurations = [
  {
    resource_name  = "cpu"
    resource_value = "1"
  },
  {
    resource_name  = "cpu_limit"
    resource_value = "0"
  },
  {
    resource_name  = "memory"
    resource_value = "10"
  },
  {
    resource_name  = "concurrency"
    resource_value = "10"
  },
  {
    resource_name  = "shortQueryConcurrencyNum"
    resource_value = "-1"
  }
]

exception_rule_name = "tf_test_exception_rule"

exception_rule_configurations = [
  {
    key   = "action"
    value = "abort"
  },
  {
    key   = "blocktime"
    value = "300"
  },
  {
    key   = "elapsedtime"
    value = "400"
  },
  {
    key   = "allcputime"
    value = "500"
  },
  {
    key   = "cpuskewpercent"
    value = "60"
  },
  {
    key   = "cpuavgpercent"
    value = "70"
  }
]
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

### 16. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the workload management resources
4. Run `terraform show` to view the created workload management resources

## Reference Information

- [Huawei Cloud Data Warehouse Service Product Documentation](https://support.huaweicloud.com/dws/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DWS Workload Management](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dws/workload-management)
