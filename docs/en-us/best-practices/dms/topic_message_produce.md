# Deploy Kafka Topic Message Produce

## Application Scenario

Distributed Message Service (DMS) for Kafka is a high-throughput, highly reliable, and highly available distributed message middleware service provided by Huawei Cloud, widely used in scenarios such as log collection, stream data processing, business decoupling, and peak shaving. In real-world business, applications need to reliably write messages to a specified Kafka topic so that downstream consumers can process them in real time.

This best practice will introduce how to use Terraform to automatically deploy a Kafka instance, create a topic, and produce messages to the topic, including the creation of VPC, subnet, security group, Kafka instance, Kafka topic, and message production.

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
- [Kafka Message Produce (huaweicloud_dms_kafka_message_produce)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_message_produce)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones.test
data.huaweicloud_dms_kafka_flavors.test
    └── huaweicloud_dms_kafka_instance.test
        └── huaweicloud_dms_kafka_topic.test
            └── huaweicloud_dms_kafka_message_produce.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_dms_kafka_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_dms_kafka_instance.test
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) article.

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones in the current region:

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zones" {
  description = "The availability zones to which the Kafka instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable availability_zones is empty, used to automatically obtain the availability zones in the current region

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
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, indicating the VPC to which the subnet belongs
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; if empty, the subnet CIDR is automatically calculated based on the VPC CIDR
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip; if empty, the gateway IP is automatically calculated based on the subnet CIDR

### 5. Create a Security Group

Add the following script in the TF file to create a security group:

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

Add the following script in the TF file to query Kafka instance flavors:

```hcl
# Query Kafka instance flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_flavor_id" {
  description = "The flavor ID of the Kafka instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_type" {
  description = "The flavor type of the Kafka instance"
  type        = string
  default     = "cluster"
}

variable "instance_storage_spec_code" {
  description = "The storage specification code of the Kafka instance"
  type        = string
  default     = "dms.physical.storage.ultra.v2"
}

data "huaweicloud_dms_kafka_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  type               = var.instance_flavor_type
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  storage_spec_code  = var.instance_storage_spec_code
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable instance_flavor_id is empty, used to automatically query Kafka instance flavors
- **type**: Assigned by referencing the input variable instance_flavor_type, indicating the flavor type
- **availability_zones**: Assigned by referencing the input variable availability_zones; if empty, the first availability zone from the availability zones data source is used
- **storage_spec_code**: Assigned by referencing the input variable instance_storage_spec_code, indicating the storage specification code

### 7. Create a Kafka Instance

Add the following script in the TF file to create a Kafka instance:

```hcl
# Create a Kafka instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the Kafka instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the Kafka instance"
  type        = string
  default     = "2.7"
}

variable "instance_storage_space" {
  description = "The storage space of the Kafka instance"
  type        = number
  default     = 600
}

variable "instance_broker_num" {
  description = "The number of brokers of the Kafka instance"
  type        = number
  default     = 3
}

variable "instance_access_user_name" {
  description = "The access user of the Kafka instance"
  type        = string
  default     = ""
}

variable "instance_access_user_password" {
  description = "The access password of the Kafka instance"
  sensitive   = true
  type        = string
  default     = ""
}

variable "instance_enabled_mechanisms" {
  description = "The enabled mechanisms of the Kafka instance"
  type        = list(string)
  default     = ["PLAIN"]
}

variable "port_protocol" {
  description = "The port protocol of the Kafka instance"

  type = object({
    private_plain_enable          = optional(bool, null)
    private_sasl_ssl_enable       = optional(bool, null)
    private_sasl_plaintext_enable = optional(bool, null)
    public_plain_enable           = optional(bool, null)
    public_sasl_ssl_enable        = optional(bool, null)
    public_sasl_plaintext_enable  = optional(bool, null)
  })

  default = {
    private_plain_enable = true
  }

  nullable = false
}

resource "huaweicloud_dms_kafka_instance" "test" {
  name               = var.instance_name
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  engine_version     = var.instance_engine_version
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_dms_kafka_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  storage_spec_code  = var.instance_storage_spec_code
  storage_space      = var.instance_storage_space
  broker_num         = var.instance_broker_num
  vpc_id             = huaweicloud_vpc.test.id
  network_id         = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  access_user        = var.instance_access_user_name
  password           = var.instance_access_user_password
  enabled_mechanisms = var.instance_enabled_mechanisms

  dynamic "port_protocol" {
    for_each = [var.port_protocol]

    content {
      private_plain_enable          = port_protocol.value.private_plain_enable
      private_sasl_ssl_enable       = port_protocol.value.private_sasl_ssl_enable
      private_sasl_plaintext_enable = port_protocol.value.private_sasl_plaintext_enable
      public_plain_enable           = port_protocol.value.public_plain_enable
      public_sasl_ssl_enable        = port_protocol.value.public_sasl_ssl_enable
      public_sasl_plaintext_enable  = port_protocol.value.public_sasl_plaintext_enable
    }
  }

  # If you want to change some of the following parameters, you need to remove the corresponding fields from "lifecycle.ignore_changes".
  lifecycle {
    ignore_changes = [
      availability_zones,
      flavor_id,
    ]
  }
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable instance_name
- **availability_zones**: Assigned by referencing the input variable availability_zones; if empty, the first availability zone from the availability zones data source is used
- **engine_version**: Assigned by referencing the input variable instance_engine_version, indicating the engine version of the instance
- **flavor_id**: Assigned by referencing the input variable instance_flavor_id; if empty, the first flavor ID from the Kafka instance flavors data source is used
- **storage_spec_code**: Assigned by referencing the input variable instance_storage_spec_code, indicating the storage specification code
- **storage_space**: Assigned by referencing the input variable instance_storage_space, indicating the storage space size
- **broker_num**: Assigned by referencing the input variable instance_broker_num, indicating the number of brokers
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, indicating the VPC to which the instance belongs
- **network_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id, indicating the subnet to which the instance belongs
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id, indicating the security group to which the instance belongs
- **access_user**: Assigned by referencing the input variable instance_access_user_name, indicating the access user of the instance
- **password**: Assigned by referencing the input variable instance_access_user_password, indicating the access password of the instance
- **enabled_mechanisms**: Assigned by referencing the input variable instance_enabled_mechanisms, indicating the enabled authentication mechanisms of the instance
- **port_protocol**: Assigned by referencing the input variable port_protocol, indicating the port protocol configuration of the instance

### 8. Create a Kafka Topic

Add the following script in the TF file to create a Kafka topic:

```hcl
# Create a Kafka topic in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "topic_name" {
  description = "The name of the topic"
  type        = string
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
  instance_id      = huaweicloud_dms_kafka_instance.test.id
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
- **instance_id**: Assigned by referencing huaweicloud_dms_kafka_instance.test.id, indicating the Kafka instance to which the topic belongs
- **name**: Assigned by referencing the input variable topic_name
- **partitions**: Assigned by referencing the input variable topic_partitions, indicating the number of partitions of the topic
- **replicas**: Assigned by referencing the input variable topic_replicas, indicating the number of replicas of the topic
- **aging_time**: Assigned by referencing the input variable topic_aging_time, indicating the aging time of the topic
- **sync_replication**: Assigned by referencing the input variable topic_sync_replication, indicating whether to enable sync replication
- **sync_flushing**: Assigned by referencing the input variable topic_sync_flushing, indicating whether to enable sync flushing
- **description**: Assigned by referencing the input variable topic_description, indicating the description of the topic
- **configs**: Assigned by referencing the input variable topic_configs, indicating the configs of the topic

### 9. Produce Kafka Topic Messages

Add the following script in the TF file to produce messages to the topic:

```hcl
# Produce Kafka topic messages in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "message_body" {
  description = "The body of the message to be sent"
  type        = string
}

variable "message_properties" {
  description = "The properties of the message to be sent"

  type = list(object({
    name  = string
    value = string
  }))

  default  = []
  nullable = false
}

resource "huaweicloud_dms_kafka_message_produce" "test" {
  instance_id = huaweicloud_dms_kafka_instance.test.id
  topic       = huaweicloud_dms_kafka_topic.test.name
  body        = var.message_body

  dynamic "property_list" {
    for_each = var.message_properties

    content {
      name  = property_list.value.name
      value = property_list.value.value
    }
  }
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing huaweicloud_dms_kafka_instance.test.id, indicating the Kafka instance to which the message belongs
- **topic**: Assigned by referencing huaweicloud_dms_kafka_topic.test.name, indicating the target topic for the message
- **body**: Assigned by referencing the input variable message_body, indicating the message content
- **property_list**: Assigned by referencing the input variable message_properties, indicating the property list of the message

### 10. Preset Input Parameters Required for Resource Deployment (Optional)

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
instance_name       = "tf_test_kafka_instance"
topic_name          = "tf_test_topic"
message_body        = "Hello Kafka!"

message_properties = [
  {
    name  = "KEY"
    value = "testKey"
  },
  {
    name  = "PARTITION"
    value = "1"
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; for other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 11. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Kafka topic message produce related resources
4. Run `terraform show` to view the created Kafka topic message produce related resources

## Reference Information

- [Huawei Cloud Distributed Message Service Kafka Product Documentation](https://support.huaweicloud.com/kafka/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DMS Kafka Topic Message Produce](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/topic-message-produce)
