# Deploy Alarm Policy

## Application Scenario

Huawei Cloud Auto Scaling service is a service that automatically adjusts computing resources, capable of automatically adjusting the number of elastic computing instances based on business needs and policies. Alarm policy is an automatic scaling policy based on monitoring metrics. By configuring CES (Cloud Eye Service) alarm rules, when monitoring metrics reach the set threshold, scaling policies are automatically triggered to perform scaling operations. This best practice will introduce how to use Terraform to automatically deploy alarm policies, including the creation of VPC network, security groups, key pairs, AS configuration, AS group, SMN topic, CES alarm rules, and alarm policies.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Compute Flavors Query Data Source (data.huaweicloud_compute_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [Images Query Data Source (data.huaweicloud_images_images)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Key Pair Resource (huaweicloud_kps_keypair)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [AS Configuration Resource (huaweicloud_as_configuration)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_configuration)
- [AS Group Resource (huaweicloud_as_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_group)
- [SMN Topic Resource (huaweicloud_smn_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [CES Alarm Rule Resource (huaweicloud_ces_alarmrule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarmrule)
- [AS Policy Resource (huaweicloud_as_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_policy)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── data.huaweicloud_images_images
            └── huaweicloud_as_configuration

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_as_group

huaweicloud_networking_secgroup
    └── huaweicloud_as_group

huaweicloud_kps_keypair
    └── huaweicloud_as_configuration

huaweicloud_as_configuration
    └── huaweicloud_as_group
        ├── huaweicloud_ces_alarmrule
        └── huaweicloud_as_policy

huaweicloud_smn_topic
    └── huaweicloud_ces_alarmrule
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones Required for Alarm Policy Resource Creation Through Data Source

Add the following script to the TF file (such as main.tf) to inform Terraform to perform a data source query, the query results are used to create alarm policy related resources:

```hcl
variable "instance_flavor_id" {
  description = "The flavor ID of the AS instance"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "The performance type of the AS instance flavor"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "The CPU core count of the AS instance flavor"
  type        = number
  default     = 2
}

variable "instance_flavor_memory_size" {
  description = "The memory size of the AS instance flavor"
  type        = number
  default     = 4
}

# Get all availability zone information in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating alarm policy related resources
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
This data source does not require additional parameters and queries all available availability zone information in the current region by default.

### 3. Query Instance Flavors Required for Alarm Policy Resource Creation Through Data Source

Add the following script to the TF file to inform Terraform to query instance flavors that meet the conditions:

```hcl
# Get all instance flavor information that meets specific conditions in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating alarm policy related resources
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**Parameter Description**:
- **count**: The number of data source creations, used to control whether to execute the instance flavor list query, only when `var.instance_flavor_id` is empty, the data source is created (i.e., execute the instance flavor list query)
- **availability_zone**: The availability zone where the instance flavor is located, using the first availability zone from the availability zone list query data source
- **performance_type**: Performance type, assigned through input variable `instance_flavor_performance_type`, default is "normal" for standard type
- **cpu_core_count**: CPU core count, assigned through input variable `instance_flavor_cpu_core_count`
- **memory_size**: Memory size (GB), assigned through input variable `instance_flavor_memory_size`

### 4. Query Images Required for Alarm Policy Resource Creation Through Data Source

Add the following script to the TF file to inform Terraform to query images that meet the conditions:

```hcl
variable "instance_image_id" {
  description = "The image ID of the AS instance"
  type        = string
  default     = ""
}

# Get all image information that meets specific conditions in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for creating alarm policy related resources
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].ids[0], null)
  visibility = "public"
  os         = "Ubuntu"
}
```

**Parameter Description**:
- **count**: The number of data source creations, used to control whether to execute the image list query, only when `var.instance_image_id` is empty, the data source is created (i.e., execute the image list query)
- **flavor_id**: The flavor ID supported by the image, if the instance flavor ID is specified, use that ID, otherwise use the first flavor ID from the instance flavor list query data source
- **visibility**: Image visibility, set to "public" for public images
- **os**: Operating system type, set to "Ubuntu"

### 5. Create VPC Resource

Add the following script to the TF file to inform Terraform to create VPC resources:

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

# Create VPC resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for providing network environment for AS group
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: The name of the VPC, assigned by referencing input variable `vpc_name`
- **cidr**: The CIDR block of the VPC, assigned by referencing input variable `vpc_cidr`, default is "192.168.0.0/16"

### 6. Create VPC Subnet Resource

Add the following script to the TF file to inform Terraform to create VPC subnet resources:

```hcl
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""

  validation {
    condition     = (var.subnet_cidr != "" && var.subnet_gateway_ip != "") || (var.subnet_cidr == "" && var.subnet_gateway_ip == "")
    error_message = "The 'subnet_cidr' and 'subnet_gateway_ip' is not allowed for only one of them to be empty"
  }
}

# Create VPC subnet resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for providing network environment for AS group
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **vpc_id**: The VPC ID to which the subnet belongs, assigned by referencing the ID of the VPC resource (huaweicloud_vpc.test)
- **name**: The name of the subnet, assigned by referencing input variable `subnet_name`
- **cidr**: The CIDR block of the subnet, if the subnet CIDR is specified, use that value, otherwise automatically calculate based on the VPC's CIDR block using the `cidrsubnet` function
- **gateway_ip**: The gateway IP of the subnet, if the gateway IP is specified, use that value, otherwise automatically calculate based on the subnet CIDR using the `cidrhost` function

### 7. Create Security Group Resource

Add the following script to the TF file to inform Terraform to create security group resources:

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# Create security group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for providing security protection for AS group
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: The name of the security group, assigned by referencing input variable `security_group_name`
- **delete_default_rules**: Whether to delete default rules, set to true to delete default rules

### 8. Create Key Pair Resource

Add the following script to the TF file to inform Terraform to create key pair resources:

```hcl
variable "keypair_name" {
  description = "The name of the key pair that is used to access the AS instance"
  type        = string
}

variable "keypair_public_key" {
  description = "The public key of the key pair that is used to access the AS instance"
  type        = string
  default     = ""
}

# Create key pair resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for accessing AS instances
resource "huaweicloud_kps_keypair" "test" {
  name       = var.keypair_name
  public_key = var.keypair_public_key != "" ? var.keypair_public_key : null
}
```

**Parameter Description**:
- **name**: The name of the key pair, assigned by referencing input variable `keypair_name`
- **public_key**: The public key of the key pair, if the public key is specified, use that value, otherwise set to null (the system will automatically generate a key pair)

### 9. Create AS Configuration Resource

Add the following script to the TF file to inform Terraform to create AS configuration resources:

```hcl
variable "configuration_name" {
  description = "The name of the AS configuration"
  type        = string
}

variable "disk_configurations" {
  description = "The disk configurations for the AS instance"
  type = list(object({
    disk_type   = string
    volume_type = string
    volume_size = number
  }))

  nullable = false

  validation {
    condition     = length(var.disk_configurations) > 0 && length([for v in var.disk_configurations : v if v.disk_type == "SYS"]) == 1
    error_message = "The 'disk_configurations' is not allowed to be empty and only one system disk is allowed"
  }
}

# Create AS configuration resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for defining templates for AS instances
resource "huaweicloud_as_configuration" "test" {
  scaling_configuration_name = var.configuration_name

  instance_config {
    image    = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
    flavor   = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
    key_name = huaweicloud_kps_keypair.test.id

    dynamic "disk" {
      for_each = var.disk_configurations

      content {
        disk_type   = disk.value.disk_type
        volume_type = disk.value.volume_type
        size        = disk.value.volume_size
      }
    }
  }
}
```

**Parameter Description**:
- **scaling_configuration_name**: The name of the AS configuration, assigned by referencing input variable `configuration_name`
- **instance_config**: Instance configuration block, defines the configuration for AS instances
  - **image**: Instance image ID, if the image ID is specified, use that value, otherwise use the first image ID from the image list query data source
  - **flavor**: Instance flavor ID, if the flavor ID is specified, use that value, otherwise use the first flavor ID from the instance flavor list query data source
  - **key_name**: Key pair ID, assigned by referencing the ID of the key pair resource (huaweicloud_kps_keypair.test)
  - **disk**: Disk configuration block, creates multiple disk configurations through dynamic block (dynamic block) based on input variable `disk_configurations`
    - **disk_type**: Disk type, assigned through `disk_type` in input variable `disk_configurations`
    - **volume_type**: Volume type, assigned through `volume_type` in input variable `disk_configurations`
    - **size**: Disk size (GB), assigned through `volume_size` in input variable `disk_configurations`

> Note: The disk configuration must include exactly one system disk (disk_type is "SYS"), and other disks are data disks.

### 10. Create AS Group Resource

Add the following script to the TF file to inform Terraform to create AS group resources:

```hcl
variable "group_name" {
  description = "The name of the AS group"
  type        = string
}

variable "desire_instance_number" {
  description = "The desired number of scaling instances in the AS group"
  type        = number
  default     = 2
}

variable "min_instance_number" {
  description = "The minimum number of scaling instances in the AS group"
  type        = number
  default     = 0
}

variable "max_instance_number" {
  description = "The maximum number of scaling instances in the AS group"
  type        = number
  default     = 10
}

variable "is_delete_publicip" {
  description = "Whether to delete the public IP address of the scaling instances when the AS group is deleted"
  type        = bool
  default     = true
}

variable "is_delete_instances" {
  description = "Whether to delete the scaling instances when the AS group is deleted"
  type        = bool
  default     = true
}

# Create AS group resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for managing AS instances
resource "huaweicloud_as_group" "test" {
  scaling_configuration_id = huaweicloud_as_configuration.test.id
  vpc_id                   = huaweicloud_vpc.test.id
  scaling_group_name       = var.group_name
  desire_instance_number   = var.desire_instance_number
  min_instance_number      = var.min_instance_number
  max_instance_number      = var.max_instance_number
  delete_publicip          = var.is_delete_publicip
  delete_instances         = var.is_delete_instances ? "yes" : "no"

  networks {
    id = huaweicloud_vpc_subnet.test.id
  }

  security_groups {
    id = huaweicloud_networking_secgroup.test.id
  }
}
```

**Parameter Description**:
- **scaling_configuration_id**: AS configuration ID, assigned by referencing the ID of the AS configuration resource (huaweicloud_as_configuration.test)
- **vpc_id**: VPC ID, assigned by referencing the ID of the VPC resource (huaweicloud_vpc.test)
- **scaling_group_name**: The name of the AS group, assigned by referencing input variable `group_name`
- **desire_instance_number**: The desired number of AS instances, assigned by referencing input variable `desire_instance_number`
- **min_instance_number**: The minimum number of AS instances, assigned by referencing input variable `min_instance_number`
- **max_instance_number**: The maximum number of AS instances, assigned by referencing input variable `max_instance_number`
- **delete_publicip**: Whether to delete the public IP address of AS instances when the AS group is deleted, assigned by referencing input variable `is_delete_publicip`
- **delete_instances**: Whether to delete AS instances when the AS group is deleted, assigned by referencing input variable `is_delete_instances`, value is "yes" or "no"
- **networks**: Network configuration block, defines the subnet used by the AS group
  - **id**: Subnet ID, assigned by referencing the ID of the VPC subnet resource (huaweicloud_vpc_subnet.test)
- **security_groups**: Security group configuration block, defines the security group used by the AS group
  - **id**: Security group ID, assigned by referencing the ID of the security group resource (huaweicloud_networking_secgroup.test)

### 11. Create SMN Topic Resource

Add the following script to the TF file to inform Terraform to create SMN topic resources:

```hcl
variable "topic_name" {
  description = "The name of the SMN topic"
  type        = string
}

# Create SMN topic resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for receiving alarm notifications
resource "huaweicloud_smn_topic" "test" {
  name = var.topic_name
}
```

**Parameter Description**:
- **name**: The name of the SMN topic, assigned by referencing input variable `topic_name`

### 12. Create CES Alarm Rule Resource

Add the following script to the TF file to inform Terraform to create CES alarm rule resources:

```hcl
variable "alarm_rule_name" {
  description = "The name of the CES alarm rule"
  type        = string
}

variable "rule_conditions" {
  description = "The conditions of the alarm rule"
  type = list(object({
    alarm_level         = optional(number, 2)
    metric_name         = string
    period              = number
    filter              = string
    comparison_operator = string
    suppress_duration   = optional(number, 0)
    value               = number
    count               = number
  }))

  nullable = false

  validation {
    condition     = length(var.rule_conditions) > 0
    error_message = "The 'rule_conditions' is not allowed to be empty"
  }
}

# Create CES alarm rule resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for monitoring AS group metrics and triggering alarms
resource "huaweicloud_ces_alarmrule" "test" {
  alarm_name = var.alarm_rule_name

  metric {
    namespace = "SYS.AS"
  }

  resources {
    dimensions {
      name  = "AutoScalingGroup"
      value = huaweicloud_as_group.test.id
    }
  }

  dynamic "condition" {
    for_each = var.rule_conditions

    content {
      alarm_level         = condition.value.alarm_level
      metric_name         = condition.value.metric_name
      period              = condition.value.period
      filter              = condition.value.filter
      comparison_operator = condition.value.comparison_operator
      suppress_duration   = condition.value.suppress_duration
      value               = condition.value.value
      count               = condition.value.count
    }
  }

  alarm_actions {
    type              = "autoscaling"
    notification_list = [huaweicloud_smn_topic.test.id]
  }
}
```

**Parameter Description**:
- **alarm_name**: The name of the alarm rule, assigned by referencing input variable `alarm_rule_name`
- **metric**: Monitoring metric configuration block
  - **namespace**: Namespace, set to "SYS.AS" for Auto Scaling service monitoring metrics
- **resources**: Monitoring resource configuration block
  - **dimensions**: Dimension configuration block, defines dimension information of monitoring resources
    - **name**: Dimension name, set to "AutoScalingGroup" for monitoring AS group
    - **value**: Dimension value, assigned by referencing the ID of the AS group resource (huaweicloud_as_group.test)
- **condition**: Alarm condition configuration block, creates multiple alarm conditions through dynamic block (dynamic block) based on input variable `rule_conditions`
  - **alarm_level**: Alarm level, assigned through `alarm_level` in input variable `rule_conditions`, default is 2 (important)
  - **metric_name**: Monitoring metric name, assigned through `metric_name` in input variable `rule_conditions`, for example "cpu_util" for CPU utilization
  - **period**: Monitoring period (seconds), assigned through `period` in input variable `rule_conditions`
  - **filter**: Aggregation method, assigned through `filter` in input variable `rule_conditions`, for example "average" for average value
  - **comparison_operator**: Comparison operator, assigned through `comparison_operator` in input variable `rule_conditions`, for example ">" for greater than
  - **suppress_duration**: Suppression duration (seconds), assigned through `suppress_duration` in input variable `rule_conditions`, default is 0
  - **value**: Threshold, assigned through `value` in input variable `rule_conditions`
  - **count**: Continuous trigger count, assigned through `count` in input variable `rule_conditions`
- **alarm_actions**: Alarm action configuration block
  - **type**: Action type, set to "autoscaling" for triggering Auto Scaling
  - **notification_list**: Notification list, assigned by referencing the ID of the SMN topic resource (huaweicloud_smn_topic.test)

### 13. Create AS Scaling Policy Resources

Add the following script to the TF file to inform Terraform to create AS scaling policy resources:

```hcl
variable "scaling_up_policy_name" {
  description = "The name of the scaling up policy"
  type        = string
}

variable "scaling_up_cool_down_time" {
  description = "The cool down time of the scaling up policy"
  type        = number
  default     = 300
}

variable "scaling_up_instance_number" {
  description = "The number of instances to add when the scaling up policy is triggered"
  type        = number
  default     = 1
}

variable "scaling_down_policy_name" {
  description = "The name of the scaling down policy"
  type        = string
}

variable "scaling_down_cool_down_time" {
  description = "The cool down time of the scaling down policy"
  type        = number
  default     = 300
}

variable "scaling_down_instance_number" {
  description = "The number of instances to remove when the scaling down policy is triggered"
  type        = number
  default     = 1
}

# Create AS scaling up policy resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for automatically scaling up when alarm is triggered
resource "huaweicloud_as_policy" "scaling_up" {
  scaling_policy_name = var.scaling_up_policy_name
  scaling_policy_type = "ALARM"
  scaling_group_id    = huaweicloud_as_group.test.id
  alarm_id            = huaweicloud_ces_alarmrule.test.id
  cool_down_time      = var.scaling_up_cool_down_time

  scaling_policy_action {
    operation       = "ADD"
    instance_number = var.scaling_up_instance_number
  }
}

# Create AS scaling down policy resource in the specified region (defaults to inheriting the region specified in the current provider block when region parameter is missing) for automatically scaling down when alarm is triggered
resource "huaweicloud_as_policy" "scaling_down" {
  scaling_policy_name = var.scaling_down_policy_name
  scaling_policy_type = "ALARM"
  scaling_group_id    = huaweicloud_as_group.test.id
  alarm_id            = huaweicloud_ces_alarmrule.test.id
  cool_down_time      = var.scaling_down_cool_down_time

  scaling_policy_action {
    operation       = "REMOVE"
    instance_number = var.scaling_down_instance_number
  }
}
```

**Parameter Description** (Scaling Up Policy):
- **scaling_policy_name**: The name of the scaling up policy, assigned by referencing input variable `scaling_up_policy_name`
- **scaling_policy_type**: Policy type, set to "ALARM" for alarm-based policy
- **scaling_group_id**: AS group ID, assigned by referencing the ID of the AS group resource (huaweicloud_as_group.test)
- **alarm_id**: Alarm rule ID, assigned by referencing the ID of the CES alarm rule resource (huaweicloud_ces_alarmrule.test)
- **cool_down_time**: Cool down time (seconds), assigned by referencing input variable `scaling_up_cool_down_time`, default is 300 seconds
- **scaling_policy_action**: Policy action configuration block
  - **operation**: Operation type, set to "ADD" for adding instances
  - **instance_number**: Instance count, assigned by referencing input variable `scaling_up_instance_number`, indicates the number of instances to add each time when scaling up

**Parameter Description** (Scaling Down Policy):
- **scaling_policy_name**: The name of the scaling down policy, assigned by referencing input variable `scaling_down_policy_name`
- **scaling_policy_type**: Policy type, set to "ALARM" for alarm-based policy
- **scaling_group_id**: AS group ID, assigned by referencing the ID of the AS group resource (huaweicloud_as_group.test)
- **alarm_id**: Alarm rule ID, assigned by referencing the ID of the CES alarm rule resource (huaweicloud_ces_alarmrule.test)
- **cool_down_time**: Cool down time (seconds), assigned by referencing input variable `scaling_down_cool_down_time`, default is 300 seconds
- **scaling_policy_action**: Policy action configuration block
  - **operation**: Operation type, set to "REMOVE" for removing instances
  - **instance_number**: Instance count, assigned by referencing input variable `scaling_down_instance_number`, indicates the number of instances to remove each time when scaling down

### 14. Preset Input Parameters Required for Resource Deployment

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, example content is as follows:

```hcl
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
keypair_name        = "tf_test_keypair"
configuration_name  = "tf_test_configuration"

disk_configurations = [
  {
    disk_type   = "SYS"
    volume_type = "SSD"
    volume_size = 40
  }
]

group_name      = "tf_test_group"
topic_name      = "tf_test_topic"
alarm_rule_name = "tf_test_alarm_rule"

rule_conditions = [
  {
    metric_name         = "cpu_util"
    period              = 300
    filter              = "average"
    comparison_operator = ">"
    value               = 80
    count               = 1
  }
]

scaling_up_policy_name   = "tf_test_scaling_up_policy"
scaling_down_policy_name = "tf_test_scaling_down_policy"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content in the `tfvars` file when executing terraform commands, other naming requires adding `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify the parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 15. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating alarm policies
4. Run `terraform show` to view the created alarm policies

## Reference Information

- [Huawei Cloud Auto Scaling Product Documentation](https://support.huaweicloud.com/as/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [AS Alarm Policy Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/as/alarm-policy)
