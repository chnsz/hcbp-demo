# Deploy Kafka Instance Configuration

## Application Scenario

Distributed Message Service (DMS) Kafka is a high-throughput, high-availability distributed message middleware widely used in scenarios such as log collection, stream data processing, service decoupling, and peak shaving. In actual deployment, a Kafka instance needs to be configured together with network resources such as Virtual Private Cloud (VPC), subnet, and security group to ensure network reachability and access security of the instance.

This best practice will introduce how to use Terraform to automatically deploy a Kafka instance configuration, including creating a VPC, subnet, and security group, and creating a Kafka instance with its flavor, storage, availability zones, and access authentication parameters configured.

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

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_dms_kafka_flavors
        └── huaweicloud_dms_kafka_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dms_kafka_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dms_kafka_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

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
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr

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
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The subnet CIDR block, automatically divided based on the VPC CIDR block when the input variable subnet_cidr is empty
- **gateway_ip**: The subnet gateway IP, automatically calculated based on the subnet CIDR block when the input variable subnet_gateway_ip is empty

### 5. Create a Security Group

Add the following script in the TF file to create a security group:

```hcl
# Create a security group in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**Parameter Description**:
- **name**: The security group name, assigned by referencing the input variable security_group_name
- **delete_default_rules**: Whether to delete the default rules of the security group, set to true to delete the default rules

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
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3)) : var.availability_zones
  storage_spec_code  = var.instance_storage_spec_code
}
```

**Parameter Description**:
- **count**: The data source is created when the input variable instance_flavor_id is empty, used to automatically query Kafka instance flavors
- **type**: The flavor type, assigned by referencing the input variable instance_flavor_type
- **availability_zones**: The availability zone list, taking the first three of the queried availability zones when the input variable availability_zones is empty
- **storage_spec_code**: The storage specification code, assigned by referencing the input variable instance_storage_spec_code

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

variable "instance_ssl_enable" {
  description = "The SSL enable of the Kafka instance"
  type        = bool
  default     = false
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

variable "instance_description" {
  description = "The description of the Kafka instance"
  type        = string
  default     = ""
}

variable "charging_mode" {
  description = "The charging mode of the Kafka instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the Kafka instance"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the Kafka instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the Kafka instance"
  type        = string
  default     = "false"
}

resource "huaweicloud_dms_kafka_instance" "test" {
  name               = var.instance_name
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3)) : var.availability_zones
  engine_version     = var.instance_engine_version
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_dms_kafka_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  storage_spec_code  = var.instance_storage_spec_code
  storage_space      = var.instance_storage_space
  broker_num         = var.instance_broker_num
  vpc_id             = huaweicloud_vpc.test.id
  network_id         = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  ssl_enable         = var.instance_ssl_enable
  access_user        = var.instance_access_user_name
  password           = var.instance_access_user_password
  description        = var.instance_description
  charging_mode      = var.charging_mode
  period_unit        = var.period_unit
  period             = var.period
  auto_renew         = var.auto_renew

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
- **name**: The Kafka instance name, assigned by referencing the input variable instance_name
- **availability_zones**: The availability zone list, taking the first three of the queried availability zones when the input variable availability_zones is empty
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version
- **flavor_id**: The flavor ID, taking the first flavor ID of the queried flavor list when the input variable instance_flavor_id is empty
- **storage_spec_code**: The storage specification code, assigned by referencing the input variable instance_storage_spec_code
- **storage_space**: The storage space, assigned by referencing the input variable instance_storage_space
- **broker_num**: The number of brokers, assigned by referencing the input variable instance_broker_num
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC created in the previous step
- **network_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet created in the previous step
- **security_group_id**: The ID of the security group to which the instance belongs, referencing the ID of the security group created in the previous step
- **ssl_enable**: Whether to enable SSL, assigned by referencing the input variable instance_ssl_enable
- **access_user**: The access user, assigned by referencing the input variable instance_access_user_name
- **password**: The access password, assigned by referencing the input variable instance_access_user_password
- **description**: The instance description, assigned by referencing the input variable instance_description
- **charging_mode**: The charging mode, assigned by referencing the input variable charging_mode
- **period_unit**: The period unit, assigned by referencing the input variable period_unit
- **period**: The period, assigned by referencing the input variable period
- **auto_renew**: Whether to enable auto renew, assigned by referencing the input variable auto_renew

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name                      = "tf_test_instance"
subnet_name                   = "tf_test_instance"
security_group_name           = "tf_test_instance"
instance_name                 = "tf_test_instance"
instance_ssl_enable           = true
instance_access_user_name     = "admin"
instance_access_user_password = "YourKafkaInstancePassword!"
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

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Kafka instance
4. Run `terraform show` to view the created Kafka instance

## Reference Information

- [Huawei Cloud Distributed Message Service Kafka Product Documentation](https://support.huaweicloud.com/kafka/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DMS Kafka Instance Configuration](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/instance-configuration)
