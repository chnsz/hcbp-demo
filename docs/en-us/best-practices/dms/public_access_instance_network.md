# Deploy Kafka Public Access Instance Network

## Application Scenario

Distributed Message Service (DMS) Kafka is a high-throughput, highly reliable message middleware service provided by Huawei Cloud, widely used in scenarios such as log collection, stream data processing, and business decoupling. By default, a Kafka instance only supports VPC private network access. When business clients reside in a local data center, another VPC, or the public network, public access must be configured for the instance.

This best practice will introduce how to use Terraform to automatically deploy a Kafka instance network configuration that supports public network access, including VPC, subnet, security group, Elastic IP (EIP), and the public access protocol configuration of the Kafka instance. By binding an EIP to each broker and enabling the corresponding public access protocols, external clients can securely connect to the Kafka instance over the public network.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Kafka Instance Flavors (data.huaweicloud_dms_kafka_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Security Group Rule (huaweicloud_networking_secgroup_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [Kafka Instance (huaweicloud_dms_kafka_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dms_kafka_instance

data.huaweicloud_dms_kafka_flavors
    └── huaweicloud_dms_kafka_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_dms_kafka_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_dms_kafka_instance

huaweicloud_vpc_eip
    └── huaweicloud_dms_kafka_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones available for the Kafka instance:

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
- **count**: Executes the query when the input variable availability_zones is empty, otherwise uses the user-specified availability zone list

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
- **cidr**: Assigned by referencing the input variable vpc_cidr, defaulting to 192.168.0.0/16

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
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id, associating with the VPC created in the previous step
- **name**: Assigned by referencing the input variable subnet_name
- **cidr**: Assigned by referencing the input variable subnet_cidr; when empty, the subnet is automatically divided based on the VPC CIDR
- **gateway_ip**: Assigned by referencing the input variable subnet_gateway_ip; when empty, the gateway IP is automatically calculated

### 5. Create a Security Group and Security Group Rule

Add the following script in the TF file to create a security group and allow the Kafka public access ports:

```hcl
# Create a security group and security group rule in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

variable "security_group_rule_ports" {
  description = "The ports of the security group rule"
  type        = string
  default     = "9094,9095"
}

variable "security_group_rule_remote_ip_prefix" {
  description = "The remote IP prefix of the security group rule"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  ports             = var.security_group_rule_ports
  remote_ip_prefix  = var.security_group_rule_remote_ip_prefix
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable security_group_name
- **delete_default_rules**: Set to true to delete the default rules of the security group
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id
- **direction**: Set to ingress, indicating an inbound rule
- **protocol**: Set to tcp, indicating the TCP protocol
- **ports**: Assigned by referencing the input variable security_group_rule_ports, defaulting to 9094,9095, which correspond to the public plaintext access port and the public encrypted access port respectively
- **remote_ip_prefix**: Assigned by referencing the input variable security_group_rule_remote_ip_prefix, specifying the client IP address or CIDR block allowed to access the Kafka instance

### 6. Query Kafka Instance Flavors

Add the following script in the TF file to query the Kafka instance flavors:

```hcl
# Query the Kafka instance flavors in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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
- **count**: Executes the query when the input variable instance_flavor_id is empty, otherwise uses the user-specified flavor ID
- **type**: Assigned by referencing the input variable instance_flavor_type, defaulting to cluster
- **availability_zones**: When the input variable availability_zones is empty, takes the first availability zone of the availability zone list, otherwise uses the user-specified availability zone list
- **storage_spec_code**: Assigned by referencing the input variable instance_storage_spec_code

### 7. Create Elastic IPs

Add the following script in the TF file to create an Elastic IP for each broker:

```hcl
# Create elastic IPs in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_broker_num" {
  description = "The number of brokers of the Kafka instance"
  type        = number
  default     = 3
}

variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "bandwidth_name" {
  description = "The name of the bandwidth"
  type        = string
}

variable "bandwidth_size" {
  description = "The size of the bandwidth"
  type        = number
  default     = 5
}

variable "bandwidth_share_type" {
  description = "The share type of the bandwidth"
  type        = string
  default     = "PER"
}

variable "bandwidth_charge_mode" {
  description = "The charge mode of the bandwidth"
  type        = string
  default     = "traffic"
}

resource "huaweicloud_vpc_eip" "test" {
  count = var.instance_broker_num

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.bandwidth_name
    size        = var.bandwidth_size
    share_type  = var.bandwidth_share_type
    charge_mode = var.bandwidth_charge_mode
  }
}
```

**Parameter Description**:
- **count**: Assigned by referencing the input variable instance_broker_num, creating one EIP for each broker; the number must match the broker count
- **publicip.type**: Assigned by referencing the input variable eip_type, defaulting to 5_bgp
- **bandwidth.name**: Assigned by referencing the input variable bandwidth_name
- **bandwidth.size**: Assigned by referencing the input variable bandwidth_size, defaulting to 5
- **bandwidth.share_type**: Assigned by referencing the input variable bandwidth_share_type, defaulting to PER
- **bandwidth.charge_mode**: Assigned by referencing the input variable bandwidth_charge_mode, defaulting to traffic

### 8. Create a Kafka Instance and Configure Public Access

Add the following script in the TF file to create a Kafka instance and enable the public access protocols:

```hcl
# Create a Kafka instance and configure public access in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

variable "instance_description" {
  description = "The description of the Kafka instance"
  type        = string
  default     = ""
}

variable "instance_access_user_name" {
  description = "The access user of the Kafka instance"
  type        = string
  default     = null
}

variable "instance_access_user_password" {
  description = "The access password of the Kafka instance"
  type        = string
  sensitive   = true
  default     = null
}

variable "instance_enabled_mechanisms" {
  description = "The enabled mechanisms of the Kafka instance"
  type        = list(string)
  default     = null
}

variable "instance_public_plain_enable" {
  description = "Whether to enable public plaintext access"
  type        = bool
  default     = true
}

variable "instance_public_sasl_ssl_enable" {
  description = "Whether to enable public SASL SSL access"
  type        = bool
  default     = false
}

variable "instance_public_sasl_plaintext_enable" {
  description = "Whether to enable public SASL plaintext access"
  type        = bool
  default     = false
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
  description        = var.instance_description
  public_ip_ids      = huaweicloud_vpc_eip.test[*].id
  access_user        = var.instance_access_user_name
  password           = var.instance_access_user_password
  enabled_mechanisms = var.instance_enabled_mechanisms

  port_protocol {
    private_plain_enable         = true
    public_plain_enable          = var.instance_public_plain_enable
    public_sasl_ssl_enable       = var.instance_public_sasl_ssl_enable
    public_sasl_plaintext_enable = var.instance_public_sasl_plaintext_enable
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
- **availability_zones**: When the input variable availability_zones is empty, takes the first three availability zones of the availability zone list, otherwise uses the user-specified availability zone list
- **engine_version**: Assigned by referencing the input variable instance_engine_version, defaulting to 2.7
- **flavor_id**: When the input variable instance_flavor_id is empty, takes the first queried flavor ID, otherwise uses the user-specified flavor ID
- **storage_spec_code**: Assigned by referencing the input variable instance_storage_spec_code
- **storage_space**: Assigned by referencing the input variable instance_storage_space, defaulting to 600
- **broker_num**: Assigned by referencing the input variable instance_broker_num, defaulting to 3
- **vpc_id**: Assigned by referencing huaweicloud_vpc.test.id
- **network_id**: Assigned by referencing huaweicloud_vpc_subnet.test.id
- **security_group_id**: Assigned by referencing huaweicloud_networking_secgroup.test.id
- **description**: Assigned by referencing the input variable instance_description
- **public_ip_ids**: Assigned by referencing huaweicloud_vpc_eip.test[*].id, binding one EIP to each broker
- **access_user**: Assigned by referencing the input variable instance_access_user_name, used for SASL authentication
- **password**: Assigned by referencing the input variable instance_access_user_password, used for SASL authentication
- **enabled_mechanisms**: Assigned by referencing the input variable instance_enabled_mechanisms; valid values are PLAIN and SCRAM-SHA-512
- **port_protocol.private_plain_enable**: Set to true to enable private plaintext access
- **port_protocol.public_plain_enable**: Assigned by referencing the input variable instance_public_plain_enable, controlling whether public plaintext access is enabled (port 9094)
- **port_protocol.public_sasl_ssl_enable**: Assigned by referencing the input variable instance_public_sasl_ssl_enable, controlling whether public SASL SSL access is enabled (port 9095)
- **port_protocol.public_sasl_plaintext_enable**: Assigned by referencing the input variable instance_public_sasl_plaintext_enable, controlling whether public SASL plaintext access is enabled (port 9094)

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name                             = "tf_test_kafka_instance"
subnet_name                          = "tf_test_kafka_instance"
security_group_name                  = "tf_test_kafka_instance"
security_group_rule_remote_ip_prefix = "your_client_ip_address"
instance_name                        = "tf_test_kafka_instance"
bandwidth_name                       = "tf_test_kafka_instance_bandwidth"
instance_access_user_name            = "admin"
instance_access_user_password        = "yourInstanceAccessPassword!"
instance_enabled_mechanisms          = ["SCRAM-SHA-512"]
instance_public_sasl_ssl_enable      = true
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

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Kafka instance that supports public access
4. Run `terraform show` to view the created Kafka instance that supports public access

## Reference Information

- [Huawei Cloud Distributed Message Service Kafka Product Documentation](https://support.huaweicloud.com/kafka/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DMS Kafka Public Access Instance Network](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/public-access-instance-network)
