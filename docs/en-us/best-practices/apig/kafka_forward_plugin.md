# Create Kafka Forward Plugin

## Application Scenario

API Gateway's Kafka forward plugin is a plugin used to asynchronously forward HTTP API requests to Kafka topics, enabling asynchronous processing of API requests and message queue integration. The Kafka forward plugin supports configuring Kafka connection information, topics, message keys, retry strategies, and other parameters, helping you achieve flexible API request forwarding management. By configuring the Kafka forward plugin, API requests can be converted into Kafka messages, enabling asynchronous processing of requests and decoupling of subsequent processing flows, improving system scalability and reliability. This best practice will introduce how to use Terraform to automatically create API Gateway's Kafka forward plugin.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DMS Kafka Flavor Query Data Source (data.huaweicloud_dms_kafka_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group Resource (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [API Gateway Instance Resource (huaweicloud_apig_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_instance)
- [DMS Kafka Instance Resource (huaweicloud_dms_kafka_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)
- [DMS Kafka Topic Resource (huaweicloud_dms_kafka_topic)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_topic)
- [API Gateway Plugin Resource (huaweicloud_apig_plugin)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/apig_plugin)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    ├── huaweicloud_apig_instance
    └── huaweicloud_dms_kafka_instance

data.huaweicloud_dms_kafka_flavors
    └── huaweicloud_dms_kafka_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_apig_instance
        └── huaweicloud_dms_kafka_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_apig_instance
    └── huaweicloud_dms_kafka_instance

huaweicloud_dms_kafka_instance
    ├── huaweicloud_dms_kafka_topic
    └── huaweicloud_apig_plugin

huaweicloud_apig_instance
    └── huaweicloud_apig_plugin
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zone Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the query results are used to create API Gateway instances and DMS Kafka instances:

```hcl
variable "availability_zones" {
  description = "The availability zones to which the instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "availability_zones_count" {
  description = "The number of availability zones to which the instance belongs"
  type        = number
  default     = 1
}

# Query all availability zone information in the specified region (defaults to the region specified in the provider block when region parameter is omitted), used to create API Gateway instances and DMS Kafka instances
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) > 0 ? 0 : 1
}
```

**Parameter Description**:
- **count**: The number of data sources to create, used to control whether to execute the availability zone list query data source, only created when `var.availability_zones` is empty (i.e., execute the availability zone list query)

### 3. Create VPC Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC resource:

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

# Create VPC resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted), used to deploy API Gateway instances and DMS Kafka instances
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr, default value is "192.168.0.0/16"

### 4. Create VPC Subnet Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a VPC subnet resource:

```hcl
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
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

# Create VPC subnet resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted), used to deploy API Gateway instances and DMS Kafka instances
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the previously created VPC resource (huaweicloud_vpc.test)
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The subnet CIDR block, assigned by referencing the input variable subnet_cidr, automatically calculated when the value is an empty string
- **gateway_ip**: The subnet gateway IP, assigned by referencing the input variable subnet_gateway_ip, automatically calculated when the value is an empty string

### 5. Create Security Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a security group resource:

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# Create security group resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted), used to deploy API Gateway instances and DMS Kafka instances
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete default rules, set to true

### 6. Create API Gateway Instance Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an API Gateway instance resource:

```hcl
variable "instance_name" {
  description = "The name of the APIG instance"
  type        = string
}

variable "instance_edition" {
  description = "The edition of the APIG instance"
  type        = string
  default     = "BASIC"
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = null
}

# Create API Gateway instance resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_apig_instance" "test" {
  name                  = var.instance_name
  edition               = var.instance_edition
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  enterprise_project_id = var.enterprise_project_id
  availability_zones    = length(var.availability_zones) > 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, var.availability_zones_count), null)
}
```

**Parameter Description**:
- **name**: The API Gateway instance name, assigned by referencing the input variable instance_name
- **edition**: The API Gateway instance edition, assigned by referencing the input variable instance_edition, default value is "BASIC"
- **vpc_id**: The VPC ID, referencing the ID of the previously created VPC resource (huaweicloud_vpc.test)
- **subnet_id**: The subnet ID, referencing the ID of the previously created VPC subnet resource (huaweicloud_vpc_subnet.test)
- **security_group_id**: The security group ID, referencing the ID of the previously created security group resource (huaweicloud_networking_secgroup.test)
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is null
- **availability_zones**: The availability zones list, uses the input variable availability_zones when it is not empty, otherwise assigned based on the return results of the availability zones query data source (data.huaweicloud_availability_zones)

### 7. Query DMS Kafka Flavor Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, the query results are used to create DMS Kafka instances:

```hcl
variable "kafka_instance_flavor_id" {
  description = "The flavor ID of the DMS Kafka instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "kafka_instance_flavor_type" {
  description = "The flavor type of the DMS Kafka instance"
  type        = string
  default     = "cluster"
}

variable "kafka_instance_storage_spec_code" {
  description = "The storage spec code of the DMS Kafka instance"
  type        = string
  default     = "dms.physical.storage.high.v2"
}

# Query DMS Kafka flavor information of the specified type in the specified region (defaults to the region specified in the provider block when region parameter is omitted), used to create DMS Kafka instances
data "huaweicloud_dms_kafka_flavors" "test" {
  count = var.kafka_instance_flavor_id != "" ? 0 : 1

  type               = var.kafka_instance_flavor_type
  storage_spec_code  = var.kafka_instance_storage_spec_code
  availability_zones = length(var.availability_zones) > 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3))
}
```

**Parameter Description**:
- **count**: The number of data sources to create, used to control whether to execute the DMS Kafka flavor query data source, only created when `var.kafka_instance_flavor_id` is empty (i.e., execute the flavor query)
- **type**: The flavor type, assigned by referencing the input variable kafka_instance_flavor_type, default value is "cluster"
- **storage_spec_code**: The storage spec code, assigned by referencing the input variable kafka_instance_storage_spec_code
- **availability_zones**: The availability zones list, uses the input variable availability_zones when it is not empty, otherwise assigned based on the return results of the availability zones query data source (data.huaweicloud_availability_zones)

### 8. Create DMS Kafka Instance Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DMS Kafka instance resource:

```hcl
variable "kafka_instance_name" {
  description = "The name of the DMS Kafka instance"
  type        = string
}

variable "kafka_instance_description" {
  description = "The description of the DMS Kafka instance"
  type        = string
  default     = ""
}

variable "kafka_instance_ssl_enable" {
  description = "Whether to enable SSL for the DMS Kafka instance"
  type        = bool
  default     = false
}

variable "kafka_instance_engine_version" {
  description = "The engine version of the DMS Kafka instance"
  type        = string
}

variable "kafka_instance_storage_space" {
  description = "The storage space of the DMS Kafka instance in GB"
  type        = number
}

variable "kafka_instance_broker_num" {
  description = "The number of brokers for the DMS Kafka instance"
  type        = number
}

variable "kafka_charging_mode" {
  description = "The charging mode of the DMS Kafka instance. Options: prePaid, postPaid"
  type        = string
  default     = "prePaid"
}

variable "kafka_period_unit" {
  description = "The period unit of the DMS Kafka instance. Options: month, year"
  type        = string
  default     = "month"
}

variable "kafka_period" {
  description = "The period of the DMS Kafka instance"
  type        = number
  default     = 1
}

variable "kafka_auto_new" {
  description = "Whether to enable auto renewal for the DMS Kafka instance"
  type        = string
  default     = "false"
}

variable "kafka_instance_user_name" {
  description = "The access user name for the DMS Kafka instance"
  type        = string
  sensitive   = true
}

variable "kafka_instance_user_password" {
  description = "The access user password for the DMS Kafka instance"
  type        = string
  sensitive   = true
}

# Create DMS Kafka instance resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_dms_kafka_instance" "test" {
  name               = var.kafka_instance_name
  description        = var.kafka_instance_description
  availability_zones = length(var.availability_zones) > 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3))
  vpc_id             = huaweicloud_vpc.test.id
  network_id         = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  ssl_enable         = var.kafka_instance_ssl_enable
  flavor_id          = var.kafka_instance_flavor_id != "" ? var.kafka_instance_flavor_id : try(data.huaweicloud_dms_kafka_flavors.test[0].flavors[0].id, null)
  engine_version     = var.kafka_instance_engine_version
  storage_spec_code  = var.kafka_instance_storage_spec_code
  storage_space      = var.kafka_instance_storage_space
  broker_num         = var.kafka_instance_broker_num
  charging_mode      = var.kafka_charging_mode
  period_unit        = var.kafka_period_unit
  period             = var.kafka_period
  auto_renew         = var.kafka_auto_new
  access_user        = var.kafka_instance_user_name
  password           = var.kafka_instance_user_password

  lifecycle {
    ignore_changes = [
      access_user,
      availability_zones,
      flavor_id,
    ]
  }
}
```

**Parameter Description**:
- **name**: The DMS Kafka instance name, assigned by referencing the input variable kafka_instance_name
- **description**: The DMS Kafka instance description, assigned by referencing the input variable kafka_instance_description, default value is an empty string
- **availability_zones**: The availability zones list, uses the input variable availability_zones when it is not empty, otherwise assigned based on the return results of the availability zones query data source (data.huaweicloud_availability_zones)
- **vpc_id**: The VPC ID, referencing the ID of the previously created VPC resource (huaweicloud_vpc.test)
- **network_id**: The network ID, referencing the ID of the previously created VPC subnet resource (huaweicloud_vpc_subnet.test)
- **security_group_id**: The security group ID, referencing the ID of the previously created security group resource (huaweicloud_networking_secgroup.test)
- **ssl_enable**: Whether to enable SSL, assigned by referencing the input variable kafka_instance_ssl_enable, default value is false
- **flavor_id**: The flavor ID, uses the input variable kafka_instance_flavor_id when it is not empty, otherwise assigned based on the return results of the DMS Kafka flavor query data source (data.huaweicloud_dms_kafka_flavors)
- **engine_version**: The engine version, assigned by referencing the input variable kafka_instance_engine_version
- **storage_spec_code**: The storage spec code, assigned by referencing the input variable kafka_instance_storage_spec_code
- **storage_space**: The storage space (unit: GB), assigned by referencing the input variable kafka_instance_storage_space
- **broker_num**: The number of brokers, assigned by referencing the input variable kafka_instance_broker_num
- **charging_mode**: The billing mode, assigned by referencing the input variable kafka_charging_mode, default value is "prePaid"
- **period_unit**: The billing period unit, assigned by referencing the input variable kafka_period_unit, default value is "month"
- **period**: The billing period, assigned by referencing the input variable kafka_period, default value is 1
- **auto_renew**: Whether to enable auto renewal, assigned by referencing the input variable kafka_auto_new, default value is "false"
- **access_user**: The access user name, assigned by referencing the input variable kafka_instance_user_name
- **password**: The access password, assigned by referencing the input variable kafka_instance_user_password

### 9. Create DMS Kafka Topic Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DMS Kafka topic resource:

```hcl
variable "kafka_topic_name" {
  description = "The name of the Kafka topic to receive messages"
  type        = string
}

variable "kafka_topic_partitions" {
  description = "The number of partitions for the Kafka topic"
  type        = number
  default     = 1
}

# Create DMS Kafka topic resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_dms_kafka_topic" "test" {
  instance_id = huaweicloud_dms_kafka_instance.test.id
  name        = var.kafka_topic_name
  partitions  = var.kafka_topic_partitions
}
```

**Parameter Description**:
- **instance_id**: The Kafka instance ID, referencing the ID of the previously created DMS Kafka instance resource (huaweicloud_dms_kafka_instance.test)
- **name**: The topic name, assigned by referencing the input variable kafka_topic_name
- **partitions**: The number of partitions, assigned by referencing the input variable kafka_topic_partitions, default value is 1

### 10. Create API Gateway Kafka Forward Plugin Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an API Gateway Kafka forward plugin resource:

```hcl
variable "plugin_name" {
  description = "The name of the Kafka forward plugin"
  type        = string
}

variable "plugin_description" {
  description = "The description of the Kafka forward plugin"
  type        = string
  default     = null
}

variable "kafka_security_protocol" {
  description = "The security protocol for Kafka connection. Options: PLAINTEXT, SASL_PLAINTEXT, SASL_SSL, SSL"
  type        = string
  default     = "PLAINTEXT"
  nullable    = false

  validation {
    condition     = contains(["PLAINTEXT", "SASL_PLAINTEXT", "SASL_SSL", "SSL"], var.kafka_security_protocol)
    error_message = "kafka_security_protocol must be one of: PLAINTEXT, SASL_PLAINTEXT, SASL_SSL, SSL."
  }
}

variable "kafka_message_key" {
  description = "The message key extraction strategy. Can be a static value or a variable expression like $context.requestId"
  type        = string
  default     = ""
}

variable "kafka_max_retry_count" {
  description = "The maximum number of retry attempts for failed message sends"
  type        = number
  default     = 3
}

variable "kafka_retry_backoff" {
  description = "The backoff time in seconds between retries"
  type        = number
  default     = 10
}

variable "kafka_sasl_mechanisms" {
  description = "The SASL mechanism for authentication. Options: PLAIN, SCRAM-SHA-256, SCRAM-SHA-512"
  type        = string
  default     = "PLAIN"

  validation {
    condition     = contains(["PLAIN", "SCRAM-SHA-256", "SCRAM-SHA-512"], var.kafka_sasl_mechanisms)
    error_message = "kafka_sasl_mechanisms must be one of: PLAIN, SCRAM-SHA-256, SCRAM-SHA-512."
  }
}

variable "kafka_sasl_username" {
  description = "The SASL username for authentication (leave empty to use kafka_access_user)"
  type        = string
  default     = ""
  sensitive   = true
  nullable    = false
}

variable "kafka_access_user" {
  description = "The access user for Kafka authentication (used when kafka_sasl_username is empty and security_protocol is not PLAINTEXT)"
  type        = string
  default     = ""
  sensitive   = true
  nullable    = false
}

variable "kafka_sasl_password" {
  description = "The SASL password for authentication (leave empty to use kafka_password)"
  type        = string
  default     = ""
  sensitive   = true
  nullable    = false
}

variable "kafka_password" {
  description = "The password for Kafka authentication (used when kafka_sasl_password is empty and security_protocol is not PLAINTEXT)"
  type        = string
  default     = ""
  sensitive   = true
  nullable    = false
}

variable "kafka_ssl_ca_content" {
  description = "The SSL CA certificate content for SSL/TLS encrypted connections"
  type        = string
  default     = ""
  sensitive   = true
  nullable    = false
}

# Create API Gateway Kafka forward plugin resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_apig_plugin" "test" {
  instance_id = huaweicloud_apig_instance.test.id
  name        = var.plugin_name
  description = var.plugin_description
  type        = "kafka_log"

  content = jsonencode({
    broker_list     = var.kafka_security_protocol == "PLAINTEXT" ? (split(",", huaweicloud_dms_kafka_instance.test.port_protocol[0].private_plain_address)) : var.kafka_security_protocol == "SASL_PLAINTEXT" ? (split(",",huaweicloud_dms_kafka_instance.test.port_protocol[0].private_sasl_plaintext_address)) : (split(",", huaweicloud_dms_kafka_instance.test.port_protocol[0].private_sasl_ssl_address))
    topic           = var.kafka_topic_name
    key             = var.kafka_message_key
    max_retry_count = var.kafka_max_retry_count
    retry_backoff   = var.kafka_retry_backoff

    sasl_config = {
      security_protocol = var.kafka_security_protocol
      sasl_mechanisms   = var.kafka_sasl_mechanisms
      sasl_username     = var.kafka_sasl_username != "" ? nonsensitive(var.kafka_sasl_username) : (var.kafka_security_protocol == "PLAINTEXT" ? "" : nonsensitive(var.kafka_access_user))
      sasl_password     = var.kafka_sasl_password != "" ? nonsensitive(var.kafka_sasl_password) : (var.kafka_security_protocol == "PLAINTEXT" ? "" : nonsensitive(var.kafka_password))
      ssl_ca_content    = var.kafka_ssl_ca_content != "" ? nonsensitive(var.kafka_ssl_ca_content) : ""
    }
  })

  lifecycle {
    ignore_changes = [
      content,
    ]
  }
}
```

**Parameter Description**:
- **instance_id**: The API Gateway instance ID, referencing the ID of the previously created API Gateway instance resource (huaweicloud_apig_instance.test)
- **name**: The plugin name, assigned by referencing the input variable plugin_name
- **description**: The plugin description, assigned by referencing the input variable plugin_description, default value is null
- **type**: The plugin type, set to "kafka_log" (indicating Kafka log forward plugin)
- **content**: The plugin configuration content, encoded as a JSON string using the jsonencode function, where:
  - **broker_list**: The Kafka Broker list, obtained from the DMS Kafka instance's port_protocol attribute based on the security protocol type
  - **topic**: The Kafka topic name, assigned by referencing the input variable kafka_topic_name
  - **key**: The message key extraction strategy, assigned by referencing the input variable kafka_message_key, can be a static value or a variable expression (e.g., $context.requestId)
  - **max_retry_count**: The maximum number of retries, assigned by referencing the input variable kafka_max_retry_count, default value is 3
  - **retry_backoff**: The retry backoff time (unit: seconds), assigned by referencing the input variable kafka_retry_backoff, default value is 10
  - **sasl_config.security_protocol**: The security protocol, assigned by referencing the input variable kafka_security_protocol
  - **sasl_config.sasl_mechanisms**: The SASL mechanism, assigned by referencing the input variable kafka_sasl_mechanisms
  - **sasl_config.sasl_username**: The SASL username, uses kafka_sasl_username when it is not empty, otherwise uses kafka_access_user when the security protocol is not PLAINTEXT
  - **sasl_config.sasl_password**: The SASL password, uses kafka_sasl_password when it is not empty, otherwise uses kafka_password when the security protocol is not PLAINTEXT
  - **sasl_config.ssl_ca_content**: The SSL CA certificate content, assigned by referencing the input variable kafka_ssl_ca_content

### 11. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# VPC Configuration
vpc_name    = "tf_test_apig_kafka_forward_plugin"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_apig_kafka_forward_plugin"

# Security Group Configuration
security_group_name = "tf_test_apig_kafka_forward_plugin"

# API Gateway Instance Configuration
instance_name    = "tf_test_apig_kafka_forward_plugin"
instance_edition = "BASIC"

# DMS Kafka Instance Configuration
kafka_instance_name              = "tf_test_apig_kafka_forward_plugin"
kafka_instance_engine_version   = "2.7"
kafka_instance_storage_space    = 600
kafka_instance_broker_num       = 3
kafka_instance_user_name         = "user"
kafka_instance_user_password     = "YourPassword123"

# DMS Kafka Topic Configuration
kafka_topic_name       = "tf_test_apig_kafka_forward_plugin"
kafka_topic_partitions = 1

# API Gateway Plugin Configuration
plugin_name            = "tf_test_apig_kafka_forward_plugin"
plugin_description     = "This is a Kafka forward plugin created by Terraform"
kafka_security_protocol = "PLAINTEXT"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=test-vpc" -var="instance_name=test-instance"`
2. Environment variables: `export TF_VAR_vpc_name=test-vpc`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 12. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the API Gateway Kafka forward plugin
4. Run `terraform show` to view the details of the created API Gateway Kafka forward plugin

## Reference Information

- [Huawei Cloud API Gateway Product Documentation](https://support.huaweicloud.com/apig/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For API Gateway Kafka Forward Plugin](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/apig/kafka-forward-plugin)
