# Deploy Event Subscription

## Application Scenario

Data Warehouse Service (DWS) is an online analytical processing (OLAP) enterprise-level data warehouse service provided by Huawei Cloud, supporting fast query and analysis of massive data. During cluster operation, DWS generates events in categories such as cluster management, security, and disaster recovery, and O&M personnel need to be aware of these events and respond in a timely manner.

This best practice will introduce how to use Terraform to automatically deploy a DWS event subscription, including the creation of a VPC, subnet, security group, DWS cluster, SMN topic and subscription, and the event subscription, helping you receive DWS cluster event notifications in a timely manner and improve O&M efficiency.

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
- [SMN Topic (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [SMN Subscription (huaweicloud_smn_subscription)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_subscription)
- [DWS Event Subscription (huaweicloud_dws_event_subscription)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_event_subscription)

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
    └── huaweicloud_dws_cluster

huaweicloud_smn_topic
    ├── huaweicloud_smn_subscription
    └── huaweicloud_dws_event_subscription
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

- **count**: The availability zones are queried only when the input variable availability_zone is empty, which is used to automatically select an availability zone for the DWS cluster

### 3. Create a Virtual Private Cloud

Add the following script in the TF file to create a VPC:

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

**Parameter description**:

- **name**: Assigned by referencing the input variable vpc_name, specifying the name of the VPC
- **cidr**: Assigned by referencing the input variable vpc_cidr, specifying the CIDR block of the VPC
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, using the default enterprise project when the variable is empty

### 4. Create a Virtual Private Cloud Subnet

Add the following script in the TF file to create a subnet:

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

**Parameter description**:

- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, specifying the VPC to which the subnet belongs
- **name**: Assigned by referencing the input variable subnet_name, specifying the name of the subnet
- **cidr**: Assigned by referencing the input variable subnet_cidr, automatically dividing the subnet CIDR block based on the VPC CIDR block when the variable is empty
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip, automatically calculating the gateway IP based on the subnet CIDR block when the variable is empty

### 5. Create a Security Group

Add the following script in the TF file to create a security group:

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

- **name**: Assigned by referencing the input variable security_group_name, specifying the name of the security group
- **delete_default_rules**: Assigned by referencing the input variable security_group_delete_default_rules, specifying whether to delete the default rules of the security group
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, using the default enterprise project when the variable is empty

### 6. Query DWS Cluster Flavors

Add the following script in the TF file to query the DWS cluster flavors:

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

- **count**: The DWS cluster flavors are queried only when the input variable cluster_node_type or cluster_version is empty
- **availability_zone**: Assigned by referencing the input variable availability_zone or the availability zone list, specifying the availability zone for flavor query
- **vcpus**: Assigned by referencing the input variable cluster_vcpus, specifying the number of vCPUs of the flavor
- **memory**: Assigned by referencing the input variable cluster_memory, specifying the memory size of the flavor
- **datastore_type**: Assigned by referencing the input variable cluster_datastore_type, specifying the datastore type of the flavor

### 7. Create a DWS Cluster

Add the following script in the TF file to create a DWS cluster:

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

- **name**: Assigned by referencing the input variable cluster_name, specifying the name of the DWS cluster
- **node_type**: Assigned by referencing the input variable cluster_node_type or the flavor list, specifying the node flavor of the cluster
- **number_of_node**: Assigned by referencing the input variable cluster_number_of_node, specifying the number of nodes in the cluster
- **number_of_cn**: Assigned by referencing the input variable cluster_number_of_cn, specifying the number of CN nodes in the cluster
- **version**: Assigned by referencing the input variable cluster_version or the flavor list, specifying the version of the cluster
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, specifying the VPC to which the cluster belongs
- **network_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id, specifying the subnet to which the cluster belongs
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id, specifying the security group to which the cluster belongs
- **availability_zone**: Assigned by referencing the input variable availability_zone or the availability zone list, specifying the availability zone of the cluster
- **user_name**: Assigned by referencing the input variable cluster_admin_user_name, specifying the administrator username of the cluster
- **user_pwd**: Assigned by referencing the input variable cluster_admin_user_pwd, specifying the administrator password of the cluster
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, using the default enterprise project when the variable is empty
- **volume.type**: Assigned by referencing the input variable cluster_volume_type, specifying the volume type of the cluster
- **volume.capacity**: Assigned by referencing the input variable cluster_volume_capacity, specifying the volume capacity of the cluster

### 8. Create an SMN Topic

Add the following script in the TF file to create an SMN topic:

```hcl
# Create an SMN topic in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "smn_topic_name" {
  description = "The name of the SMN topic"
  type        = string
}

variable "smn_topic_display_name" {
  description = "The display name of the SMN topic"
  type        = string
  default     = ""
}

resource "huaweicloud_smn_topic" "test" {
  name         = var.smn_topic_name
  display_name = var.smn_topic_display_name
}
```

**Parameter description**:

- **name**: Assigned by referencing the input variable smn_topic_name, specifying the name of the SMN topic
- **display_name**: Assigned by referencing the input variable smn_topic_display_name, specifying the display name of the SMN topic

### 9. Create an SMN Subscription

Add the following script in the TF file to create an SMN subscription:

```hcl
# Create an SMN subscription in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "smn_subscription_endpoint" {
  description = "The message endpoint"
  type        = string
}

variable "smn_subscription_protocol" {
  description = "The protocol of the message endpoint"
  type        = string
}

variable "smn_subscription_remark" {
  description = "The remark information"
  type        = string
  default     = null
}

resource "huaweicloud_smn_subscription" "test" {
  topic_urn = huaweicloud_smn_topic.test.id
  endpoint  = var.smn_subscription_endpoint
  protocol  = var.smn_subscription_protocol
  remark    = var.smn_subscription_remark
}
```

**Parameter description**:

- **topic_urn**: Assigned by referencing huaweicloud_smn_topic.test.id, specifying the SMN topic to which the subscription belongs
- **endpoint**: Assigned by referencing the input variable smn_subscription_endpoint, specifying the message receiving endpoint
- **protocol**: Assigned by referencing the input variable smn_subscription_protocol, specifying the message receiving protocol
- **remark**: Assigned by referencing the input variable smn_subscription_remark, specifying the remark information of the subscription

### 10. Create a DWS Event Subscription

Add the following script in the TF file to create a DWS event subscription:

```hcl
# Create a DWS event subscription in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "event_subscription_name" {
  description = "The name of the DWS event subscription"
  type        = string
}

variable "event_category" {
  description = "The event categories to subscribe"
  type        = string
}

variable "event_severity" {
  description = "The event severities to subscribe"
  type        = string
}

variable "event_source_type" {
  description = "The event source types to subscribe"
  type        = string
}

variable "time_zone" {
  description = "The time zone for alarm and event subscriptions"
  type        = string
  default     = "GMT+08:00"
}

resource "huaweicloud_dws_event_subscription" "test" {
  name                     = var.event_subscription_name
  enable                   = "1"
  notification_target      = huaweicloud_smn_topic.test.id
  notification_target_name = huaweicloud_smn_topic.test.name
  notification_target_type = "SMN"
  category                 = var.event_category
  severity                 = var.event_severity
  source_type              = var.event_source_type
  time_zone                = var.time_zone
}
```

**Parameter description**:

- **name**: Assigned by referencing the input variable event_subscription_name, specifying the name of the event subscription
- **enable**: Specifies whether to enable the event subscription, where 1 means enabled
- **notification_target**: Assigned by referencing huaweicloud_smn_topic.test.id, specifying the target of the event notification
- **notification_target_name**: Assigned by referencing huaweicloud_smn_topic.test.name, specifying the name of the event notification target
- **notification_target_type**: Specifies the type of the event notification target, which currently only supports SMN
- **category**: Assigned by referencing the input variable event_category, specifying the event categories to subscribe
- **severity**: Assigned by referencing the input variable event_severity, specifying the event severities to subscribe
- **source_type**: Assigned by referencing the input variable event_source_type, specifying the event source types to subscribe
- **time_zone**: Assigned by referencing the input variable time_zone, specifying the time zone of the event subscription

### 11. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name                  = "tf_test_dws_vpc"
vpc_cidr                  = "192.168.0.0/16"
subnet_name               = "tf_test_dws_subnet"
security_group_name       = "tf_test_dws_sg"
cluster_name              = "tf_test_cluster"
cluster_admin_user_name   = "dbadmin"
cluster_admin_user_pwd    = "YourPassword@123"
smn_topic_name            = "tf_test_dws_topic"
smn_subscription_endpoint = "mailtest@gmail.com"
smn_subscription_protocol = "email"
event_subscription_name   = "tf_test_dws_event"
event_category            = "management,security"
event_severity            = "normal,warning"
event_source_type         = "cluster,disaster-recovery"
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

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DWS event subscription
4. Run `terraform show` to view the created DWS event subscription

## Reference Information

- [Huawei Cloud Data Warehouse Service Product Documentation](https://support.huaweicloud.com/dws/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DWS Event Subscription](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dws/event-subscription)
