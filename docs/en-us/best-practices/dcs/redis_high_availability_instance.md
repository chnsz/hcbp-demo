# Deploy Redis High Availability Instance

## Application Scenario

Distributed Cache Service (DCS) is a high-performance, highly available in-memory database service provided by Huawei Cloud, supporting mainstream cache engines such as Redis and Memcached. A Redis high availability instance uses deployment modes such as master-standby, cluster, proxy cluster, or read/write splitting to automatically fail over when a single node fails, ensuring cache service continuity and data reliability. It is suitable for business scenarios with high availability requirements, such as session storage, hot data caching, and message queues.

This best practice will introduce how to use Terraform to automatically deploy a DCS Redis high availability instance, including VPC and subnet creation, automatic query of availability zones and flavors, instance creation, and configuration of backup policy, whitelists, parameters, and tags.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS Flavors (data.huaweicloud_dcs_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [DCS Instance (huaweicloud_dcs_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a VPC

Add the following script in the TF file (such as main.tf) to create a virtual private cloud:

```hcl
# Create a virtual private cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr

### 3. Create a VPC Subnet

Add the following script in the TF file (such as main.tf) to create a virtual private cloud subnet:

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
  description = "The gateway IP address of the subnet"
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
- **vpc_id**: The ID of the VPC to which the subnet belongs, referencing the ID of the VPC resource created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet, assigned by referencing the input variable subnet_cidr; if not specified, the subnet is automatically divided based on the VPC CIDR block
- **gateway_ip**: The gateway IP address of the subnet, assigned by referencing the input variable subnet_gateway_ip; if not specified, it is automatically calculated based on the subnet CIDR block

### 4. Query the Availability Zones

Add the following script in the TF file (such as main.tf) to query the availability zones:

```hcl
# Query the availability zones data source in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "availability_zones" {
  description = "The availability zones to which the Redis instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) < 1 ? 1 : 0
}
```

**Parameter description**:
- **count**: The data source query is executed only when the input variable availability_zones does not specify availability zones

### 5. Query the DCS Flavors

Add the following script in the TF file (such as main.tf) to query the DCS flavors:

```hcl
# Query the DCS flavors data source in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_flavor_id" {
  description = "The flavor ID of the Redis instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_cache_mode" {
  description = "The cache mode of the Redis instance"
  type        = string
  default     = "ha"
}

variable "instance_capacity" {
  description = "The capacity of the Redis instance (in GB)"
  type        = number
  default     = 4
}

variable "instance_engine_version" {
  description = "The engine version of the Redis instance"
  type        = string
  default     = "5.0"
}

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  cache_mode     = var.instance_cache_mode
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**Parameter description**:
- **count**: The data source query is executed only when the input variable instance_flavor_id does not specify a flavor
- **cache_mode**: The cache mode, assigned by referencing the input variable instance_cache_mode
- **capacity**: The instance capacity (in GB), assigned by referencing the input variable instance_capacity
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version

### 6. Create a DCS Redis Instance

Add the following script in the TF file (such as main.tf) to create a DCS Redis instance:

```hcl
# Create a DCS instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "instance_name" {
  description = "The name of the Redis instance"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the Redis instance belongs"
  type        = string
  default     = null
}

variable "instance_password" {
  description = "The password for the Redis instance"
  type        = string
  sensitive   = true
  default     = null
}

variable "instance_backup_policy" {
  description = "The backup policy of the Redis instance"
  type        = object({
    backup_type = optional(string, "auto")
    backup_at   = list(number)
    begin_at    = string
    save_days   = optional(number, null)
    period_type = optional(string, null)
  })
  default     = null
}

variable "instance_whitelists" {
  description = "The whitelists of the Redis instance"
  type        = list(object({
    group_name = string
    ip_address = list(string)
  }))
  default     = []
  nullable    = false
}

variable "instance_parameters" {
  description = "The parameters of the Redis instance"
  type        = list(object({
    id    = string
    name  = string
    value = string
  }))
  default     = []
  nullable    = false
}

variable "instance_tags" {
  description = "The tags of the Redis instance"
  type        = map(string)
  default     = {}
}

variable "instance_rename_commands" {
  description = "The rename commands of the Redis instance"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the Redis instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The unit of the period"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the Redis instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "Whether auto renew is enabled"
  type        = string
  default     = "false"
}

resource "huaweicloud_dcs_instance" "test" {
  name                  = var.instance_name
  engine                = "Redis"
  enterprise_project_id = var.enterprise_project_id
  engine_version        = var.instance_engine_version
  capacity              = var.instance_capacity
  flavor                = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_dcs_flavors.test[0].flavors[0].name, null)
  availability_zones    = length(var.availability_zones) > 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 2), null)
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  password              = var.instance_password

  dynamic "backup_policy" {
    for_each = var.instance_backup_policy != null ? [var.instance_backup_policy] : []

    content {
      backup_type = lookup(backup_policy.value, "backup_type", null)
      save_days   = lookup(backup_policy.value, "save_days", null)
      period_type = lookup(backup_policy.value, "period_type", null)
      backup_at   = backup_policy.value["backup_at"]
      begin_at    = backup_policy.value["begin_at"]
    }
  }

  dynamic "whitelists" {
    for_each = var.instance_whitelists

    content {
      group_name = whitelists.value["group_name"]
      ip_address = whitelists.value["ip_address"]
    }
  }

  dynamic "parameters" {
    for_each = var.instance_parameters

    content {
      id    = parameters.value["id"]
      name  = parameters.value["name"]
      value = parameters.value["value"]
    }
  }

  tags            = var.instance_tags
  rename_commands = var.instance_rename_commands

  charging_mode = var.charging_mode
  period_unit   = var.period_unit
  period        = var.period
  auto_renew    = var.auto_renew

  # If you want to change the `flavor` or `availability_zones`, you need to delete it from the `lifecycle.ignore_changes`.
  lifecycle {
    ignore_changes = [
      flavor,
      availability_zones,
    ]
  }
}
```

**Parameter description**:
- **name**: The instance name, assigned by referencing the input variable instance_name
- **engine**: The cache engine, fixed to Redis
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **engine_version**: The engine version, assigned by referencing the input variable instance_engine_version
- **capacity**: The instance capacity (in GB), assigned by referencing the input variable instance_capacity
- **flavor**: The instance flavor, using the input variable instance_flavor_id first, and referencing the DCS flavors data source query result when not specified
- **availability_zones**: The availability zone list, using the input variable availability_zones first, and referencing the first two zones of the availability zones data source query result when not specified
- **vpc_id**: The ID of the VPC to which the instance belongs, referencing the ID of the VPC resource created in the previous step
- **subnet_id**: The ID of the subnet to which the instance belongs, referencing the ID of the subnet resource created in the previous step
- **password**: The instance password, assigned by referencing the input variable instance_password
- **backup_policy**: The backup policy, assigned by referencing the input variable instance_backup_policy
- **whitelists**: The whitelists, assigned by referencing the input variable instance_whitelists
- **parameters**: The instance parameters, assigned by referencing the input variable instance_parameters
- **tags**: The instance tags, assigned by referencing the input variable instance_tags
- **rename_commands**: The rename commands, assigned by referencing the input variable instance_rename_commands
- **charging_mode**: The charging mode, assigned by referencing the input variable charging_mode
- **period_unit**: The period unit, assigned by referencing the input variable period_unit
- **period**: The period, assigned by referencing the input variable period
- **auto_renew**: Whether auto renew is enabled, assigned by referencing the input variable auto_renew

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name               = "tf_test_dcs_instance_vpc"
subnet_name            = "tf_test_dcs_instance_subnet"
instance_name          = "tf_test_dcs_instance"
instance_password      = "YourRedisInstancePassword!"
instance_backup_policy = {
  backup_type = "auto"
  backup_at   = [1, 3, 4, 5, 6]
  begin_at    = "02:00-04:00"
  save_days   = 7
}

instance_whitelists = [
  {
    group_name = "test-group1"
    ip_address = ["192.168.10.100", "192.168.0.0/24"]
  },
  {
    group_name = "test-group2"
    ip_address = ["172.16.10.100", "172.16.0.0/24"]
  }
]
instance_parameters = [
  {
    id    = "1"
    name  = "timeout"
    value = "500"
  },
  {
    id    = "3"
    name  = "hash-max-ziplist-entries"
    value = "4096"
  }
]

instance_tags = {
  foo = "bar"
}
```

**Usage method**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DCS Redis high availability instance
4. Run `terraform show` to view the created DCS Redis high availability instance

## Reference Information

- [Huawei Cloud DCS Product Documentation](https://support.huaweicloud.com/dcs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DCS Redis High Availability Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-high-availability-instance)
