# Deploy RabbitMQ Message Producer and Consumer

## Application Scenario

Distributed Message Service (DMS) for RabbitMQ is a highly available, highly reliable, and high-performance message middleware service provided by Huawei Cloud, fully compatible with open-source RabbitMQ and supporting message sending and receiving, message storage, and message routing. In microservice and distributed systems, producers and consumers communicate asynchronously through message queues to achieve system decoupling and peak shaving, which is a common pattern for building loosely coupled architectures.

This best practice will introduce how to use Terraform to automatically deploy a complete RabbitMQ message producer and consumer scenario, including creating a VPC, subnet, security group, RabbitMQ instance, virtual host, exchange, queue, and the binding between the exchange and the queue, and deploying producer and consumer applications on two ECS instances to automatically send and consume messages.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RabbitMQ Instance Flavors (data.huaweicloud_dms_rabbitmq_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_rabbitmq_flavors)
- [ECS Flavors (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [Images (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RabbitMQ Instance (huaweicloud_dms_rabbitmq_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_instance)
- [RabbitMQ Virtual Host (huaweicloud_dms_rabbitmq_vhost)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_vhost)
- [RabbitMQ Exchange (huaweicloud_dms_rabbitmq_exchange)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_exchange)
- [RabbitMQ Queue (huaweicloud_dms_rabbitmq_queue)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_queue)
- [RabbitMQ Exchange Associate (huaweicloud_dms_rabbitmq_exchange_associate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_rabbitmq_exchange_associate)
- [ECS Instance (huaweicloud_compute_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_dms_rabbitmq_flavors
    ├── data.huaweicloud_compute_flavors
    └── huaweicloud_dms_rabbitmq_instance
        └── huaweicloud_dms_rabbitmq_vhost
            ├── huaweicloud_dms_rabbitmq_exchange
            ├── huaweicloud_dms_rabbitmq_queue
            └── huaweicloud_dms_rabbitmq_exchange_associate

data.huaweicloud_images_images
    ├── huaweicloud_compute_instance.producer
    └── huaweicloud_compute_instance.consumer

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_dms_rabbitmq_instance
        ├── huaweicloud_compute_instance.producer
        └── huaweicloud_compute_instance.consumer

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    ├── huaweicloud_dms_rabbitmq_instance
    ├── huaweicloud_compute_instance.producer
    └── huaweicloud_compute_instance.consumer
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones in the current region:

```hcl
# Query the availability zones in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
data "huaweicloud_availability_zones" "test" {}
```

### 3. Query RabbitMQ Instance Flavors

Add the following script in the TF file (such as main.tf) to query the RabbitMQ instance flavors:

```hcl
# Query the RabbitMQ instance flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_flavor_type" {
  description = "The flavor type of the RabbitMQ instance"
  type        = string
  default     = "cluster"
}

variable "instance_storage_spec_code" {
  description = "The storage specification code of the RabbitMQ instance"
  type        = string
  default     = "dms.physical.storage.ultra.v2"
}

data "huaweicloud_dms_rabbitmq_flavors" "test" {
  type               = var.instance_flavor_type
  storage_spec_code  = var.instance_storage_spec_code
  availability_zones = try(slice(data.huaweicloud_availability_zones.test.names, 0, 1), null)
}
```

**Parameter Description**:
- **type**: Assigned by referencing the input variable instance_flavor_type, indicating the flavor type of the RabbitMQ instance
- **storage_spec_code**: Assigned by referencing the input variable instance_storage_spec_code, indicating the storage specification code of the RabbitMQ instance
- **availability_zones**: Assigned by referencing the availability zones data source, indicating the availability zones of the RabbitMQ instance

### 4. Query ECS Flavors and Images

Add the following script in the TF file (such as main.tf) to query the ECS flavors and images used for deploying the producer and consumer applications:

```hcl
# Query the ECS flavors and images in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "ecs_image_name" {
  description = "The image name of the ECS instances (Ubuntu)"
  type        = string
  default     = "Ubuntu 20.04 server 64bit"
}

data "huaweicloud_compute_flavors" "test" {
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = "normal"
  cpu_core_count    = 2
  memory_size       = 4
}

data "huaweicloud_images_images" "test" {
  name       = var.ecs_image_name
  visibility = "public"
}
```

**Parameter Description**:
- **availability_zone**: Assigned by referencing the availability zones data source, indicating the availability zone of the ECS instance
- **performance_type**: Indicates the performance type of the ECS flavor, which is normal here
- **cpu_core_count**: Indicates the number of CPU cores of the ECS flavor, which is 2 here
- **memory_size**: Indicates the memory size of the ECS flavor, which is 4 here
- **name**: Assigned by referencing the input variable ecs_image_name, indicating the name of the ECS image
- **visibility**: Indicates the visibility of the image, which is public here

### 5. Create a VPC

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
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable vpc_name, indicating the name of the VPC
- **cidr**: Assigned by referencing the input variable vpc_cidr, indicating the CIDR block of the VPC

### 6. Create a Subnet

Add the following script in the TF file (such as main.tf) to create a subnet:

```hcl
# Create a subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = "192.168.0.0/24"
  nullable    = true
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = "192.168.0.1"
  nullable    = true
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **vpc_id**: Assigned by referencing the VPC resource, indicating the VPC to which the subnet belongs
- **name**: Assigned by referencing the input variable subnet_name, indicating the name of the subnet
- **cidr**: Assigned by referencing the input variable subnet_cidr, which is automatically calculated based on the VPC CIDR block if empty
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip, which is automatically calculated based on the subnet CIDR block if empty

### 7. Create a Security Group and Security Group Rules

Add the following script in the TF file (such as main.tf) to create a security group and its rules:

```hcl
# Create a security group and security group rules in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

variable "security_group_rule_configurations" {
  description = "The list of security group rule configurations."

  type = list(object({
    direction        = string
    ethertype        = string
    protocol         = string
    port_range_min   = number
    port_range_max   = number
    remote_ip_prefix = string
    description      = string
  }))

  default = [
    {
      direction        = "ingress"
      ethertype        = "IPv4"
      protocol         = "tcp"
      port_range_min   = 5672
      port_range_max   = 5672
      remote_ip_prefix = "192.168.0.0/16"
      description      = "Allow ECS instances to access RabbitMQ"
    },
    {
      direction        = "ingress"
      ethertype        = "IPv4"
      protocol         = "tcp"
      port_range_min   = 22
      port_range_max   = 22
      remote_ip_prefix = "192.168.0.0/16"
      description      = "Allow SSH access"
    }
  ]
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = length(var.security_group_rule_configurations)

  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = lookup(var.security_group_rule_configurations[count.index], "direction", null)
  ethertype         = lookup(var.security_group_rule_configurations[count.index], "ethertype", null)
  protocol          = lookup(var.security_group_rule_configurations[count.index], "protocol", null)
  port_range_min    = lookup(var.security_group_rule_configurations[count.index], "port_range_min", null)
  port_range_max    = lookup(var.security_group_rule_configurations[count.index], "port_range_max", null)
  remote_ip_prefix  = lookup(var.security_group_rule_configurations[count.index], "remote_ip_prefix", null)
  description       = lookup(var.security_group_rule_configurations[count.index], "description", null)
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable security_group_name, indicating the name of the security group
- **security_group_id**: Assigned by referencing the security group resource, indicating the security group to which the security group rule belongs
- **direction**: Assigned by referencing the input variable security_group_rule_configurations, indicating the direction of the security group rule
- **ethertype**: Assigned by referencing the input variable security_group_rule_configurations, indicating the IP protocol type of the security group rule
- **protocol**: Assigned by referencing the input variable security_group_rule_configurations, indicating the protocol type of the security group rule
- **port_range_min**: Assigned by referencing the input variable security_group_rule_configurations, indicating the minimum port range of the security group rule
- **port_range_max**: Assigned by referencing the input variable security_group_rule_configurations, indicating the maximum port range of the security group rule
- **remote_ip_prefix**: Assigned by referencing the input variable security_group_rule_configurations, indicating the remote IP address of the security group rule
- **description**: Assigned by referencing the input variable security_group_rule_configurations, indicating the description of the security group rule

### 8. Create a RabbitMQ Instance

Add the following script in the TF file (such as main.tf) to create a RabbitMQ instance:

```hcl
# Create a RabbitMQ instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the RabbitMQ instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the RabbitMQ instance"
  type        = string
  default     = "3.8.35"
}

variable "instance_broker_num" {
  description = "The number of brokers of the RabbitMQ instance"
  type        = number
  default     = 3
}

variable "instance_storage_space" {
  description = "The storage space of the RabbitMQ instance"
  type        = number
  default     = 600
}

variable "instance_ssl_enable" {
  description = "Whether to enable SSL for the RabbitMQ instance"
  type        = bool
  default     = false
}

variable "instance_access_user_name" {
  description = "The access user of the RabbitMQ instance"
  type        = string
  default     = "admin"
}

variable "instance_password" {
  description = "The access password of the RabbitMQ instance"
  type        = string
  sensitive   = true
  default     = "123456"
}

variable "instance_description" {
  description = "The description of the RabbitMQ instance"
  type        = string
  default     = ""
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the RabbitMQ instance belongs"
  type        = string
  default     = null
}

variable "instance_tags" {
  description = "The key/value pairs to associate with the RabbitMQ instance"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the RabbitMQ instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the RabbitMQ instance"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the RabbitMQ instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the RabbitMQ instance"
  type        = string
  default     = "false"
}

resource "huaweicloud_dms_rabbitmq_instance" "test" {
  name                  = var.instance_name
  engine_version        = var.instance_engine_version
  flavor_id             = try(data.huaweicloud_dms_rabbitmq_flavors.test.flavors[0].id, null)
  vpc_id                = huaweicloud_vpc.test.id
  network_id            = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zones    = try([data.huaweicloud_availability_zones.test.names[0]], null)
  broker_num            = var.instance_broker_num
  storage_space         = var.instance_storage_space
  storage_spec_code     = var.instance_storage_spec_code
  ssl_enable            = var.instance_ssl_enable
  access_user           = var.instance_access_user_name
  password              = var.instance_password
  description           = var.instance_description
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.instance_tags
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew

  lifecycle {
    ignore_changes = [
      flavor_id,
      availability_zones,
    ]
  }
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable instance_name, indicating the name of the RabbitMQ instance
- **engine_version**: Assigned by referencing the input variable instance_engine_version, indicating the engine version of the RabbitMQ instance
- **flavor_id**: Assigned by referencing the RabbitMQ instance flavors data source, indicating the flavor of the RabbitMQ instance
- **vpc_id**: Assigned by referencing the VPC resource, indicating the VPC to which the RabbitMQ instance belongs
- **network_id**: Assigned by referencing the subnet resource, indicating the subnet to which the RabbitMQ instance belongs
- **security_group_id**: Assigned by referencing the security group resource, indicating the security group bound to the RabbitMQ instance
- **availability_zones**: Assigned by referencing the availability zones data source, indicating the availability zones of the RabbitMQ instance
- **broker_num**: Assigned by referencing the input variable instance_broker_num, indicating the number of brokers of the RabbitMQ instance
- **storage_space**: Assigned by referencing the input variable instance_storage_space, indicating the storage space of the RabbitMQ instance
- **storage_spec_code**: Assigned by referencing the input variable instance_storage_spec_code, indicating the storage specification code of the RabbitMQ instance
- **ssl_enable**: Assigned by referencing the input variable instance_ssl_enable, indicating whether to enable SSL
- **access_user**: Assigned by referencing the input variable instance_access_user_name, indicating the access user of the RabbitMQ instance
- **password**: Assigned by referencing the input variable instance_password, indicating the access password of the RabbitMQ instance
- **description**: Assigned by referencing the input variable instance_description, indicating the description of the RabbitMQ instance
- **enterprise_project_id**: Assigned by referencing the input variable enterprise_project_id, indicating the enterprise project to which the RabbitMQ instance belongs
- **tags**: Assigned by referencing the input variable instance_tags, indicating the tags of the RabbitMQ instance
- **charging_mode**: Assigned by referencing the input variable charging_mode, indicating the charging mode of the RabbitMQ instance
- **period_unit**: Assigned by referencing the input variable period_unit, indicating the period unit of the RabbitMQ instance
- **period**: Assigned by referencing the input variable period, indicating the period of the RabbitMQ instance
- **auto_renew**: Assigned by referencing the input variable auto_renew, indicating whether the RabbitMQ instance is automatically renewed

### 9. Create a RabbitMQ Virtual Host

Add the following script in the TF file (such as main.tf) to create a RabbitMQ virtual host:

```hcl
# Create a RabbitMQ virtual host in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vhost_name" {
  description = "The name of the RabbitMQ virtual host"
  type        = string
  default     = "app_vhost"
}

resource "huaweicloud_dms_rabbitmq_vhost" "test" {
  instance_id = huaweicloud_dms_rabbitmq_instance.test.id
  name        = var.vhost_name
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the RabbitMQ instance resource, indicating the RabbitMQ instance to which the virtual host belongs
- **name**: Assigned by referencing the input variable vhost_name, indicating the name of the virtual host

### 10. Create a RabbitMQ Exchange

Add the following script in the TF file (such as main.tf) to create a RabbitMQ exchange:

```hcl
# Create a RabbitMQ exchange in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "exchange_name" {
  description = "The name of the RabbitMQ exchange"
  type        = string
  default     = "app_exchange"
}

variable "exchange_type" {
  description = "The type of the RabbitMQ exchange"
  type        = string
  default     = "direct"
}

resource "huaweicloud_dms_rabbitmq_exchange" "test" {
  instance_id = huaweicloud_dms_rabbitmq_instance.test.id
  vhost       = var.vhost_name
  name        = var.exchange_name
  type        = var.exchange_type
  auto_delete = false
  durable     = true
  internal    = false

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_dms_rabbitmq_vhost.test
  ]
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the RabbitMQ instance resource, indicating the RabbitMQ instance to which the exchange belongs
- **vhost**: Assigned by referencing the input variable vhost_name, indicating the virtual host to which the exchange belongs
- **name**: Assigned by referencing the input variable exchange_name, indicating the name of the exchange
- **type**: Assigned by referencing the input variable exchange_type, indicating the type of the exchange
- **auto_delete**: Indicates whether the exchange is automatically deleted, which is false here
- **durable**: Indicates whether the exchange is durable, which is true here
- **internal**: Indicates whether the exchange is internal, which is false here

### 11. Create a RabbitMQ Queue

Add the following script in the TF file (such as main.tf) to create a RabbitMQ queue:

```hcl
# Create a RabbitMQ queue in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "queue_name" {
  description = "The name of the RabbitMQ queue"
  type        = string
  default     = "app_queue"
}

resource "huaweicloud_dms_rabbitmq_queue" "test" {
  instance_id = huaweicloud_dms_rabbitmq_instance.test.id
  vhost       = var.vhost_name
  name        = var.queue_name
  auto_delete = false
  durable     = true

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_dms_rabbitmq_vhost.test
  ]
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the RabbitMQ instance resource, indicating the RabbitMQ instance to which the queue belongs
- **vhost**: Assigned by referencing the input variable vhost_name, indicating the virtual host to which the queue belongs
- **name**: Assigned by referencing the input variable queue_name, indicating the name of the queue
- **auto_delete**: Indicates whether the queue is automatically deleted, which is false here
- **durable**: Indicates whether the queue is durable, which is true here

### 12. Create a RabbitMQ Exchange Associate

Add the following script in the TF file (such as main.tf) to bind the exchange to the queue:

```hcl
# Create a RabbitMQ exchange associate in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_dms_rabbitmq_exchange_associate" "test" {
  instance_id      = huaweicloud_dms_rabbitmq_instance.test.id
  vhost            = var.vhost_name
  exchange         = var.exchange_name
  destination_type = "Queue"
  destination      = var.queue_name
  routing_key      = var.queue_name

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_dms_rabbitmq_vhost.test,
    huaweicloud_dms_rabbitmq_exchange.test,
    huaweicloud_dms_rabbitmq_queue.test
  ]
}
```

**Parameter Description**:
- **instance_id**: Assigned by referencing the RabbitMQ instance resource, indicating the RabbitMQ instance to which the associate belongs
- **vhost**: Assigned by referencing the input variable vhost_name, indicating the virtual host to which the associate belongs
- **exchange**: Assigned by referencing the input variable exchange_name, indicating the name of the exchange to be bound
- **destination_type**: Indicates the destination type of the associate, which is Queue here
- **destination**: Assigned by referencing the input variable queue_name, indicating the name of the destination queue to be bound
- **routing_key**: Assigned by referencing the input variable queue_name, indicating the routing key of the associate

### 13. Create a Producer ECS Instance

Add the following script in the TF file (such as main.tf) to create a producer ECS instance and automatically deploy the producer application through user_data:

```hcl
# Create a producer ECS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "producer_instance_name" {
  description = "The name of the producer ECS instance"
  type        = string
}

variable "eip_type" {
  description = "The type of the ECS EIP"
  type        = string
  default     = "5_bgp"
}

variable "eip_share_type" {
  description = "The share type of the ECS EIP"
  type        = string
  default     = "PER"
}

variable "eip_size" {
  description = "The size of the ECS EIP"
  type        = number
  default     = 5
}

variable "eip_charge_mode" {
  description = "The charge mode of the ECS EIP"
  type        = string
  default     = "traffic"
}

variable "message_interval" {
  description = "The interval in seconds between messages sent by the producer"
  type        = number
  default     = 5
}

resource "huaweicloud_compute_instance" "producer" {
  name              = var.producer_instance_name
  image_id          = try(data.huaweicloud_images_images.test.images[0].id, null)
  flavor_id         = try(data.huaweicloud_compute_flavors.test.flavors[0].id, null)
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  admin_pass        = var.instance_password
  eip_type          = var.eip_type

  security_group_ids = [huaweicloud_networking_secgroup.test.id]

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  bandwidth {
    share_type  = var.eip_share_type
    size        = var.eip_size
    charge_mode = var.eip_charge_mode
  }

  user_data = templatefile("${path.module}/templates/user_data_producer_tpl", {
    producer_script   = file("${path.module}/apps/producer.py")
    rabbitmq_host     = huaweicloud_dms_rabbitmq_instance.test.connect_address
    rabbitmq_user     = var.instance_access_user_name
    rabbitmq_password = var.instance_password
    rabbitmq_vhost    = var.vhost_name
    queue_name        = var.queue_name
    exchange_name     = var.exchange_name
    exchange_type     = var.exchange_type
    routing_key       = var.queue_name
    message_interval  = var.message_interval
  })

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_dms_rabbitmq_vhost.test,
    huaweicloud_dms_rabbitmq_queue.test,
    huaweicloud_dms_rabbitmq_exchange_associate.test
  ]

  lifecycle {
    ignore_changes = [
      image_id,
      flavor_id,
      availability_zone
    ]
  }
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable producer_instance_name, indicating the name of the producer ECS instance
- **image_id**: Assigned by referencing the images data source, indicating the image of the producer ECS instance
- **flavor_id**: Assigned by referencing the ECS flavors data source, indicating the flavor of the producer ECS instance
- **availability_zone**: Assigned by referencing the availability zones data source, indicating the availability zone of the producer ECS instance
- **admin_pass**: Assigned by referencing the input variable instance_password, indicating the login password of the producer ECS instance
- **eip_type**: Assigned by referencing the input variable eip_type, indicating the EIP type of the producer ECS instance
- **security_group_ids**: Assigned by referencing the security group resource, indicating the security group bound to the producer ECS instance
- **network**: Indicates the network configuration of the producer ECS instance, assigned by referencing the subnet resource
- **bandwidth**: Indicates the bandwidth configuration of the producer ECS instance, assigned by referencing the input variables eip_share_type, eip_size, and eip_charge_mode
- **user_data**: Indicates the initialization script of the producer ECS instance, used to automatically deploy the producer application

### 14. Create a Consumer ECS Instance

Add the following script in the TF file (such as main.tf) to create a consumer ECS instance and automatically deploy the consumer application through user_data:

```hcl
# Create a consumer ECS instance in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "consumer_instance_name" {
  description = "The name of the consumer ECS instance"
  type        = string
}

resource "huaweicloud_compute_instance" "consumer" {
  name              = var.consumer_instance_name
  image_id          = try(data.huaweicloud_images_images.test.images[0].id, null)
  flavor_id         = try(data.huaweicloud_compute_flavors.test.flavors[0].id, null)
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  admin_pass        = var.instance_password
  eip_type          = var.eip_type

  security_group_ids = [huaweicloud_networking_secgroup.test.id]

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  bandwidth {
    share_type  = var.eip_share_type
    size        = var.eip_size
    charge_mode = var.eip_charge_mode
  }

  user_data = templatefile("${path.module}/templates/user_data_consumer_tpl", {
    consumer_script   = file("${path.module}/apps/consumer.py")
    rabbitmq_host     = huaweicloud_dms_rabbitmq_instance.test.connect_address
    rabbitmq_user     = var.instance_access_user_name
    rabbitmq_password = var.instance_password
    rabbitmq_vhost    = var.vhost_name
    queue_name        = var.queue_name
  })

  depends_on = [
    huaweicloud_dms_rabbitmq_instance.test,
    huaweicloud_dms_rabbitmq_vhost.test,
    huaweicloud_dms_rabbitmq_queue.test,
    huaweicloud_dms_rabbitmq_exchange_associate.test
  ]

  lifecycle {
    ignore_changes = [
      image_id,
      flavor_id,
      availability_zone
    ]
  }
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable consumer_instance_name, indicating the name of the consumer ECS instance
- **image_id**: Assigned by referencing the images data source, indicating the image of the consumer ECS instance
- **flavor_id**: Assigned by referencing the ECS flavors data source, indicating the flavor of the consumer ECS instance
- **availability_zone**: Assigned by referencing the availability zones data source, indicating the availability zone of the consumer ECS instance
- **admin_pass**: Assigned by referencing the input variable instance_password, indicating the login password of the consumer ECS instance
- **eip_type**: Assigned by referencing the input variable eip_type, indicating the EIP type of the consumer ECS instance
- **security_group_ids**: Assigned by referencing the security group resource, indicating the security group bound to the consumer ECS instance
- **network**: Indicates the network configuration of the consumer ECS instance, assigned by referencing the subnet resource
- **bandwidth**: Indicates the bandwidth configuration of the consumer ECS instance, assigned by referencing the input variables eip_share_type, eip_size, and eip_charge_mode
- **user_data**: Indicates the initialization script of the consumer ECS instance, used to automatically deploy the consumer application

### 15. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Network variables
vpc_name            = "tf_test_vpc_rabbitmq"
subnet_name         = "tf_test_subnet_rabbitmq"
security_group_name = "tf_test_sg_rabbitmq"

# RabbitMQ instance variables
instance_name             = "tf_test_rabbitmq_instance"
instance_access_user_name = "admin"
instance_password         = "YourPassword@123"

# ECS instance variables
producer_instance_name = "tf_test_producer"
consumer_instance_name = "tf_test_consumer"
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
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the RabbitMQ message producer and consumer scenario
4. Run `terraform show` to view the created RabbitMQ message producer and consumer scenario

## Reference Information

- [Huawei Cloud Distributed Message Service RabbitMQ Product Documentation](https://support.huaweicloud.com/rabbitmq/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DMS RabbitMQ Message Producer and Consumer](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/rabbitmq/message-producer-and-consumer)
