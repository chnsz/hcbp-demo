# Deploy Event Subscription

## Application Scenario

Data Warehouse Service (DWS) is an online analytical processing (OLAP) database service provided by Huawei Cloud, supporting fast query and analysis of massive data. In daily O&M, key events of a cluster, such as scaling, restart, and failures, need to be perceived in time so that O&M personnel can respond quickly and ensure business continuity and stability.

This best practice will introduce how to use Terraform to automatically deploy a DWS event subscription, including the creation of a VPC, subnet, security group, DWS cluster, SMN topic and subscription, and DWS event subscription, helping you quickly build the event notification capability of a DWS cluster through Infrastructure as Code (IaC).

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
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Virtual Private Cloud

Add the following script in the TF file (such as main.tf) to create a VPC:

```hcl
# Create a virtual private cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id; an empty string means the default enterprise project is used

### 3. Create a Virtual Private Cloud Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create a virtual private cloud subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC resource created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet, assigned by referencing the input variable subnet_cidr; an empty string means the subnet is automatically divided based on the VPC CIDR block
- **gateway_ip**: The gateway IP of the subnet, assigned by referencing the input variable subnet_gateway_ip; an empty string means the gateway IP is automatically calculated based on the subnet CIDR block

### 4. Create a Security Group

Add the following script in the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group, assigned by referencing the input variable security_group_delete_default_rules
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id; an empty string means the default enterprise project is used

### 5. Query the Availability Zone List

Add the following script in the TF file (such as main.tf) to query the availability zone list:

```hcl
# Query the availability zone list data source in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **count**: The number of data sources to create; when the input variable availability_zone is an empty string, this data source is created to automatically obtain the availability zone list in the current region

### 6. Query DWS Cluster Flavors

Add the following script in the TF file (such as main.tf) to query DWS cluster flavors:

```hcl
# Query the DWS cluster flavors data source in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **count**: The number of data sources to create; when the input variable cluster_node_type or cluster_version is an empty string, this data source is created to automatically obtain the matching cluster flavor
- **availability_zone**: The availability zone, assigned by referencing the input variable availability_zone; an empty string means the first availability zone returned by the availability zone list data source is used
- **vcpus**: The number of vCPUs of the cluster node, assigned by referencing the input variable cluster_vcpus
- **memory**: The memory size of the cluster node, assigned by referencing the input variable cluster_memory
- **datastore_type**: The datastore type of the cluster, assigned by referencing the input variable cluster_datastore_type

### 7. Create a DWS Cluster

Add the following script in the TF file (such as main.tf) to create a DWS cluster:

```hcl
# Create a DWS cluster resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **name**: The DWS cluster name, assigned by referencing the input variable cluster_name
- **node_type**: The cluster node flavor, assigned by referencing the input variable cluster_node_type; an empty string means the flavor ID returned by the DWS cluster flavors data source is used
- **number_of_node**: The number of cluster nodes, assigned by referencing the input variable cluster_number_of_node
- **number_of_cn**: The number of CN nodes in the cluster, assigned by referencing the input variable cluster_number_of_cn
- **version**: The cluster version, assigned by referencing the input variable cluster_version; an empty string means the version returned by the DWS cluster flavors data source is used
- **vpc_id**: The ID of the VPC to which the cluster belongs, referencing the ID of the VPC resource created in the previous step
- **network_id**: The ID of the subnet to which the cluster belongs, referencing the ID of the subnet resource created in the previous step
- **security_group_id**: The ID of the security group to which the cluster belongs, referencing the ID of the security group resource created in the previous step
- **availability_zone**: The availability zone of the cluster, assigned by referencing the input variable availability_zone; an empty string means the first availability zone returned by the availability zone list data source is used
- **user_name**: The administrator username of the cluster, assigned by referencing the input variable cluster_admin_user_name
- **user_pwd**: The administrator password of the cluster, assigned by referencing the input variable cluster_admin_user_pwd
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id; an empty string means the default enterprise project is used
- **volume.type**: The data disk type of the cluster, assigned by referencing the input variable cluster_volume_type
- **volume.capacity**: The data disk capacity of the cluster, assigned by referencing the input variable cluster_volume_capacity

### 8. Create an SMN Topic

Add the following script in the TF file (such as main.tf) to create an SMN topic:

```hcl
# Create an SMN topic resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:
- **name**: The SMN topic name, assigned by referencing the input variable smn_topic_name
- **display_name**: The display name of the SMN topic, assigned by referencing the input variable smn_topic_display_name

### 9. Create an SMN Subscription

Add the following script in the TF file (such as main.tf) to create an SMN subscription:

```hcl
# Create an SMN subscription resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:
- **topic_urn**: The URN of the SMN topic to which the subscription belongs, referencing the ID of the SMN topic resource created in the previous step
- **endpoint**: The message receiving endpoint, assigned by referencing the input variable smn_subscription_endpoint
- **protocol**: The message receiving protocol, assigned by referencing the input variable smn_subscription_protocol
- **remark**: The subscription remark information, assigned by referencing the input variable smn_subscription_remark

### 10. Create a DWS Event Subscription

Add the following script in the TF file (such as main.tf) to create a DWS event subscription:

```hcl
# Create a DWS event subscription resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:
- **name**: The event subscription name, assigned by referencing the input variable event_subscription_name
- **enable**: Whether to enable the event subscription; the value "1" means enabled
- **notification_target**: The notification target, referencing the ID of the SMN topic resource created in the previous step
- **notification_target_name**: The notification target name, referencing the name of the SMN topic resource created in the previous step
- **notification_target_type**: The notification target type, currently only "SMN" is supported
- **category**: The event categories to subscribe, assigned by referencing the input variable event_category
- **severity**: The event severities to subscribe, assigned by referencing the input variable event_severity
- **source_type**: The event source types to subscribe, assigned by referencing the input variable event_source_type
- **time_zone**: The time zone of the event subscription, assigned by referencing the input variable time_zone

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

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
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
