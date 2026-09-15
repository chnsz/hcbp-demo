# Deploy Kafka Instance Data Replication

## Application Scenario

Distributed Message Service (DMS) for Kafka provides high-throughput and highly reliable message middleware capabilities, and is widely used in scenarios such as log collection, stream data processing, and service decoupling. When data needs to be synchronized between different Kafka instances, Smart Connect tasks can be used to replicate data between instances, meeting requirements such as cross-instance data migration, active-active disaster recovery, and data aggregation.

This best practice will introduce how to use Terraform to automatically deploy Kafka instance data replication, including the creation of a VPC, subnet, security group, multiple Kafka instances, a Kafka topic, Smart Connect, and a Smart Connect task.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Kafka Instance Flavors (data.huaweicloud_dms_kafka_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Kafka Instance (huaweicloud_dms_kafka_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)
- [Kafka Topic (huaweicloud_dms_kafka_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_topic)
- [Kafka Smart Connect (huaweicloud_dms_kafka_smart_connect)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_smart_connect)
- [Kafka Smart Connect Task (huaweicloud_dms_kafkav2_smart_connect_task)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafkav2_smart_connect_task)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dms_kafka_instance

data.huaweicloud_dms_kafka_flavors
    └── huaweicloud_dms_kafka_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dms_kafka_instance
            ├── huaweicloud_dms_kafka_topic
            ├── huaweicloud_dms_kafka_smart_connect
            └── huaweicloud_dms_kafkav2_smart_connect_task

huaweicloud_networking_secgroup
    └── huaweicloud_dms_kafka_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf):

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_configurations" {
  description = "The list of configurations for multiple Kafka instances"

  type = list(object({
    name               = string
    availability_zones = optional(list(string), [])
    engine_version     = optional(string, "3.x")
    flavor_id          = optional(string, "")
    flavor_type        = optional(string, "cluster")
    storage_spec_code  = optional(string, "dms.physical.storage.ultra.v2")
    storage_space      = optional(number, 600)
    broker_num         = optional(number, 3)
    access_user        = optional(string, "")
    password           = optional(string, "")
    enabled_mechanisms = optional(list(string), null)

    port_protocol = optional(object({
      private_plain_enable          = optional(bool, true)
      private_sasl_ssl_enable       = optional(bool, null)
      private_sasl_plaintext_enable = optional(bool, null)
    }), {})
  }))

  nullable = false
  default  = []

  validation {
    condition     = length(var.instance_configurations) >= 2
    error_message = "At least two instances are required"
  }
}

data "huaweicloud_availability_zones" "test" {
  count = anytrue([for v in var.instance_configurations : length(v.availability_zones) == 0]) ? 1 : 0
}
```

**Parameter Description**:
- **count**: The availability zones are queried only when any Kafka instance configuration does not specify availability zones, so that availability zones can be automatically assigned to the instances

### 3. Create a VPC

Add the following script in the TF file (such as main.tf):

```hcl
# Create a VPC in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr

### 4. Create a VPC Subnet

Add the following script in the TF file (such as main.tf):

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
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing the ID of the resource huaweicloud_vpc.test
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; when not specified, the subnet CIDR is automatically calculated based on the VPC CIDR
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip; when not specified, the gateway IP is automatically calculated based on the subnet CIDR

### 5. Create a Security Group

Add the following script in the TF file (such as main.tf):

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable security_group_name

### 6. Query Kafka Instance Flavors

Add the following script in the TF file (such as main.tf):

```hcl
# Query the Kafka instance flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
locals {
  instance_configurations_without_flavor_id = [for v in var.instance_configurations : v if v.flavor_id == ""]
}

data "huaweicloud_dms_kafka_flavors" "test" {
  count = length(local.instance_configurations_without_flavor_id)

  type               = local.instance_configurations_without_flavor_id[count.index].flavor_type
  availability_zones = length(local.instance_configurations_without_flavor_id[count.index].availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : null
  storage_spec_code  = local.instance_configurations_without_flavor_id[count.index].storage_spec_code
}
```

**Parameter Description**:
- **count**: The number of Kafka instance configurations without a specified flavor ID
- **type**: Assigned by referencing the flavor_type in the instance configuration
- **availability_zones**: Assigned by referencing the result of the availability zones data source
- **storage_spec_code**: Assigned by referencing the storage_spec_code in the instance configuration

### 7. Create Kafka Instances

Add the following script in the TF file (such as main.tf):

```hcl
# Create Kafka instances in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_dms_kafka_instance" "test" {
  count = length(var.instance_configurations)

  name               = var.instance_configurations[count.index].name
  availability_zones = length(var.instance_configurations[count.index].availability_zones) > 0 ? var.instance_configurations[count.index].availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1))
  engine_version     = var.instance_configurations[count.index].engine_version
  flavor_id          = var.instance_configurations[count.index].flavor_id != "" ? var.instance_configurations[count.index].flavor_id : try(data.huaweicloud_dms_kafka_flavors.test[count.index].flavors[0].id, null)
  storage_spec_code  = var.instance_configurations[count.index].storage_spec_code
  storage_space      = var.instance_configurations[count.index].storage_space
  broker_num         = var.instance_configurations[count.index].broker_num
  vpc_id             = huaweicloud_vpc.test.id
  network_id         = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  access_user        = var.instance_configurations[count.index].access_user
  password           = var.instance_configurations[count.index].password
  enabled_mechanisms = var.instance_configurations[count.index].enabled_mechanisms

  dynamic "port_protocol" {
    for_each = length(var.instance_configurations[count.index].port_protocol) > 0 ? [var.instance_configurations[count.index].port_protocol] : []

    content {
      private_plain_enable          = port_protocol.value.private_plain_enable
      private_sasl_ssl_enable       = port_protocol.value.private_sasl_ssl_enable
      private_sasl_plaintext_enable = port_protocol.value.private_sasl_plaintext_enable
    }
  }

  lifecycle {
    ignore_changes = [
      availability_zones,
      flavor_id,
    ]
  }
}
```

**Parameter Description**:
- **count**: The number of Kafka instance configurations, at least 2
- **name**: Assigned by referencing the name in the instance configuration
- **availability_zones**: Assigned by referencing the availability_zones in the instance configuration or the result of the availability zones data source
- **engine_version**: Assigned by referencing the engine_version in the instance configuration
- **flavor_id**: Assigned by referencing the flavor_id in the instance configuration or the result of the Kafka instance flavors data source
- **storage_spec_code**: Assigned by referencing the storage_spec_code in the instance configuration
- **storage_space**: Assigned by referencing the storage_space in the instance configuration
- **broker_num**: Assigned by referencing the broker_num in the instance configuration
- **vpc_id**: Assigned by referencing the ID of the resource huaweicloud_vpc.test
- **network_id**: Assigned by referencing the ID of the resource huaweicloud_vpc_subnet.test
- **security_group_id**: Assigned by referencing the ID of the resource huaweicloud_networking_secgroup.test
- **access_user**: Assigned by referencing the access_user in the instance configuration
- **password**: Assigned by referencing the password in the instance configuration
- **enabled_mechanisms**: Assigned by referencing the enabled_mechanisms in the instance configuration
- **port_protocol**: Assigned by referencing the port_protocol in the instance configuration, used to configure the port protocol of the instance

### 8. Create a Kafka Topic

Add the following script in the TF file (such as main.tf):

```hcl
# Create a Kafka topic in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "task_topics" {
  description = "The topics of the Smart Connect task"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "topic_name" {
  description = "The name of the Kafka topic"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = var.topic_name != "" || length(var.task_topics) > 0
    error_message = "topic_name is required when task_topics is not provided"
  }
}

variable "topic_partitions" {
  description = "The number of partitions of the topic"
  type        = number
  default     = 10
}

variable "topic_replicas" {
  description = "The number of replicas of the topic"
  type        = number
  default     = 3
}

variable "topic_aging_time" {
  description = "The aging time of the topic"
  type        = number
  default     = 72
}

variable "topic_sync_replication" {
  description = "The sync replication of the topic"
  type        = bool
  default     = false
}

variable "topic_sync_flushing" {
  description = "The sync flushing of the topic"
  type        = bool
  default     = false
}

variable "topic_description" {
  description = "The description of the topic"
  type        = string
  default     = null
}

variable "topic_configs" {
  description = "The configs of the topic"

  type = list(object({
    name  = string
    value = string
  }))

  default  = []
  nullable = false
}

resource "huaweicloud_dms_kafka_topic" "test" {
  count = length(var.task_topics) == 0 ? 1 : 0

  instance_id      = huaweicloud_dms_kafka_instance.test[0].id
  name             = var.topic_name
  partitions       = var.topic_partitions
  replicas         = var.topic_replicas
  aging_time       = var.topic_aging_time
  sync_replication = var.topic_sync_replication
  sync_flushing    = var.topic_sync_flushing
  description      = var.topic_description

  dynamic "configs" {
    for_each = var.topic_configs

    content {
      name  = configs.value.name
      value = configs.value.value
    }
  }
}
```

**Parameter Description**:
- **count**: The topic is created when no Smart Connect task topic is specified
- **instance_id**: Assigned by referencing the ID of the resource huaweicloud_dms_kafka_instance.test[0]
- **name**: Assigned by referencing the input variable topic_name
- **partitions**: Assigned by referencing the input variable topic_partitions
- **replicas**: Assigned by referencing the input variable topic_replicas
- **aging_time**: Assigned by referencing the input variable topic_aging_time
- **sync_replication**: Assigned by referencing the input variable topic_sync_replication
- **sync_flushing**: Assigned by referencing the input variable topic_sync_flushing
- **description**: Assigned by referencing the input variable topic_description
- **configs**: Assigned by referencing the input variable topic_configs, used to configure topic parameters

### 9. Create Kafka Smart Connect

Add the following script in the TF file (such as main.tf):

```hcl
# Create Kafka Smart Connect in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "smart_connect_storage_spec_code" {
  description = "The storage specification code of the Smart Connect"
  type        = string
  default     = null
}

variable "smart_connect_bandwidth" {
  description = "The bandwidth of the Smart Connect"
  type        = string
  default     = null
}

variable "smart_connect_node_count" {
  description = "The number of nodes of the Smart Connect"
  type        = number
  default     = 2
}

resource "huaweicloud_dms_kafka_smart_connect" "test" {
  instance_id       = huaweicloud_dms_kafka_instance.test[0].id
  storage_spec_code = var.smart_connect_storage_spec_code
  bandwidth         = var.smart_connect_bandwidth
  node_count        = var.smart_connect_node_count
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the ID of the resource huaweicloud_dms_kafka_instance.test[0]
- **storage_spec_code**: Assigned by referencing the input variable smart_connect_storage_spec_code
- **bandwidth**: Assigned by referencing the input variable smart_connect_bandwidth
- **node_count**: Assigned by referencing the input variable smart_connect_node_count

### 10. Create a Kafka Smart Connect Task

Add the following script in the TF file (such as main.tf):

```hcl
# Create a Kafka Smart Connect task in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "task_name" {
  description = "The name of the Smart Connect task"
  type        = string
}

variable "task_start_later" {
  description = "The start later of the Smart Connect task"
  type        = bool
  default     = false
}

variable "task_direction" {
  description = "The direction of the Smart Connect task"
  type        = string
  default     = "two-way"
}

variable "task_replication_factor" {
  description = "The replication factor of the Smart Connect task"
  type        = number
  default     = 3
}

variable "task_task_num" {
  description = "The number of tasks of the Smart Connect task"
  type        = number
  default     = 2
}

variable "task_provenance_header_enabled" {
  description = "The provenance header enabled of the Smart Connect task"
  type        = bool
  default     = false
}

variable "task_sync_consumer_offsets_enabled" {
  description = "The sync consumer offsets enabled of the Smart Connect task"
  type        = bool
  default     = false
}

variable "task_rename_topic_enabled" {
  description = "The rename topic enabled of the Smart Connect task"
  type        = bool
  default     = true
}

variable "task_consumer_strategy" {
  description = "The consumer strategy of the Smart Connect task"
  type        = string
  default     = "latest"
}

variable "task_compression_type" {
  description = "The compression type of the Smart Connect task"
  type        = string
  default     = "none"
}

variable "task_topics_mapping" {
  description = "The topics mapping of the Smart Connect task"
  type        = list(string)
  default     = []
}

resource "huaweicloud_dms_kafkav2_smart_connect_task" "test" {
  instance_id = huaweicloud_dms_kafka_instance.test[0].id
  task_name   = var.task_name
  source_type = "KAFKA_REPLICATOR_SOURCE"
  start_later = var.task_start_later
  topics      = length(var.task_topics) > 0 ? var.task_topics : huaweicloud_dms_kafka_topic.test[*].name

  source_task {
    peer_instance_id              = huaweicloud_dms_kafka_instance.test[1].id
    direction                     = var.task_direction
    replication_factor            = var.task_replication_factor
    task_num                      = var.task_task_num
    provenance_header_enabled     = var.task_provenance_header_enabled
    sync_consumer_offsets_enabled = var.task_sync_consumer_offsets_enabled
    rename_topic_enabled          = var.task_rename_topic_enabled
    consumer_strategy             = var.task_consumer_strategy
    compression_type              = var.task_compression_type
    topics_mapping                = var.task_topics_mapping
    security_protocol             = try(huaweicloud_dms_kafka_instance.test[1].port_protocol[0].private_sasl_ssl_enable, false) ? "SASL_SSL" : try(huaweicloud_dms_kafka_instance.test[1].port_protocol[0].private_sasl_plaintext_enable, false) ? "PLAINTEXT" : null
    sasl_mechanism                = try(tolist(huaweicloud_dms_kafka_instance.test[1].enabled_mechanisms)[0], null)
    user_name                     = try(huaweicloud_dms_kafka_instance.test[1].access_user, null)
    password                      = try(huaweicloud_dms_kafka_instance.test[1].password, null)
  }

  depends_on = [huaweicloud_dms_kafka_smart_connect.test]
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the ID of the resource huaweicloud_dms_kafka_instance.test[0]
- **task_name**: Assigned by referencing the input variable task_name
- **source_type**: The source type of the Smart Connect task, fixed to KAFKA_REPLICATOR_SOURCE
- **start_later**: Assigned by referencing the input variable task_start_later
- **topics**: Assigned by referencing the input variable task_topics or the name of the resource huaweicloud_dms_kafka_topic.test
- **peer_instance_id**: Assigned by referencing the ID of the resource huaweicloud_dms_kafka_instance.test[1]
- **direction**: Assigned by referencing the input variable task_direction
- **replication_factor**: Assigned by referencing the input variable task_replication_factor
- **task_num**: Assigned by referencing the input variable task_task_num
- **provenance_header_enabled**: Assigned by referencing the input variable task_provenance_header_enabled
- **sync_consumer_offsets_enabled**: Assigned by referencing the input variable task_sync_consumer_offsets_enabled
- **rename_topic_enabled**: Assigned by referencing the input variable task_rename_topic_enabled
- **consumer_strategy**: Assigned by referencing the input variable task_consumer_strategy
- **compression_type**: Assigned by referencing the input variable task_compression_type
- **topics_mapping**: Assigned by referencing the input variable task_topics_mapping
- **security_protocol**: Automatically selected based on the port protocol of the peer Kafka instance
- **sasl_mechanism**: Automatically assigned based on the SASL mechanism of the peer Kafka instance
- **user_name**: Automatically assigned based on the access user of the peer Kafka instance
- **password**: Automatically assigned based on the access password of the peer Kafka instance

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
vpc_name            = "tf_test_kafka_instance"
subnet_name         = "tf_test_kafka_instance"
security_group_name = "tf_test_kafka_instance"
task_name           = "tf_test_kafka_task"
topic_name          = "tf_test_kafka_topic"

instance_configurations = [
  {
    name = "tf_test_instance"
  },
  {
    name               = "tf_test_peer_instance"
    access_user        = "admin"
    password           = "YourKafkaInstancePassword!"
    enabled_mechanisms = ["SCRAM-SHA-512"]
    port_protocol = {
      private_plain_enable    = false
      private_sasl_ssl_enable = true
    }
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

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Kafka instance data replication related resources
4. Run `terraform show` to view the created Kafka instance data replication related resources

## Reference Information

- [Huawei Cloud Distributed Message Service Kafka Product Documentation](https://support.huaweicloud.com/kafka/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DMS Kafka Instance Data Replication](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/replicate-instance-data)
