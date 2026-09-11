# Deploy Queue Public Network Connectivity

## Application Scenario

DLI queues are deployed within a user VPC by default and cannot directly access data sources or services on the public network. When a business needs a DLI queue to access the public network (for example, to call external APIs, pull public data, or interact with public services), you need to configure an enhanced datasource connection for the queue and use a NAT gateway SNAT rule to provide the public network egress.

This best practice will introduce how to use Terraform to automatically deploy DLI queue public network connectivity, including the creation and association of an elastic resource pool, a queue, a VPC and subnet, an enhanced datasource connection, an EIP, a NAT gateway, and an SNAT rule.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DLI Elastic Resource Pool (huaweicloud_dli_elastic_resource_pool)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_elastic_resource_pool)
- [DLI Queue (huaweicloud_dli_queue)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_queue)
- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [DLI Enhanced Datasource Connection (huaweicloud_dli_datasource_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_datasource_connection)
- [DLI Enhanced Datasource Connection Associate (huaweicloud_dli_datasource_connection_associate)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_datasource_connection_associate)
- [Elastic IP (huaweicloud_vpc_eip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [NAT Gateway (huaweicloud_nat_gateway)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_gateway)
- [NAT SNAT Rule (huaweicloud_nat_snat_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_snat_rule)

### Resource/Data Source Dependencies

```
huaweicloud_dli_elastic_resource_pool
    ├── huaweicloud_dli_queue
    │   └── huaweicloud_dli_datasource_connection_associate
    ├── huaweicloud_dli_datasource_connection_associate
    └── huaweicloud_nat_snat_rule

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    │   ├── huaweicloud_dli_datasource_connection
    │   └── huaweicloud_nat_gateway
    ├── huaweicloud_dli_datasource_connection
    └── huaweicloud_nat_gateway

huaweicloud_vpc_eip
    └── huaweicloud_nat_snat_rule
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a DLI Elastic Resource Pool

Add the following script in the TF file (such as main.tf) to create a DLI elastic resource pool:

```hcl
# Create a DLI elastic resource pool in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "elastic_resource_pool_name" {
  description = "The name of the DLI elastic resource pool"
  type        = string
}

variable "elastic_resource_pool_description" {
  description = "The description of the elastic resource pool"
  type        = string
  default     = ""
}

variable "elastic_resource_pool_min_cu" {
  description = "The minimum number of CUs for the elastic resource pool"
  type        = number
  default     = 16
}

variable "elastic_resource_pool_max_cu" {
  description = "The maximum number of CUs for the elastic resource pool"
  type        = number
  default     = 64
}

variable "elastic_resource_pool_cidr" {
  description = "The CIDR block of the elastic resource pool. This CIDR must not overlap with the VPC CIDR"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_dli_elastic_resource_pool" "test" {
  name                  = var.elastic_resource_pool_name
  description           = var.elastic_resource_pool_description
  min_cu                = var.elastic_resource_pool_min_cu
  max_cu                = var.elastic_resource_pool_max_cu
  cidr                  = var.elastic_resource_pool_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  label = {
    spec = "basic"
  }
}
```

**Parameter Description**:
- **name**: The elastic resource pool name, assigned by referencing the input variable elastic_resource_pool_name
- **description**: The elastic resource pool description, assigned by referencing the input variable elastic_resource_pool_description
- **min_cu**: The minimum number of CUs for the elastic resource pool, assigned by referencing the input variable elastic_resource_pool_min_cu
- **max_cu**: The maximum number of CUs for the elastic resource pool, assigned by referencing the input variable elastic_resource_pool_max_cu
- **cidr**: The CIDR block of the elastic resource pool, assigned by referencing the input variable elastic_resource_pool_cidr. This CIDR must not overlap with the VPC CIDR
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id

### 3. Create a DLI Queue

Add the following script in the TF file to create a DLI queue:

```hcl
# Create a DLI queue in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "queue_name" {
  description = "The name of the DLI exclusive queue"
  type        = string
}

variable "queue_type" {
  description = "The type of the DLI queue. The valid values are sql and general"
  type        = string
  default     = "sql"

  validation {
    condition     = contains(["sql", "general"], var.queue_type)
    error_message = "The queue_type valid value must be `sql` or `general`."
  }
}

variable "queue_cu_count" {
  description = "The CU count of the DLI queue"
  type        = number
  default     = 16
}

variable "queue_description" {
  description = "The description of the DLI queue"
  type        = string
  default     = ""
}

resource "huaweicloud_dli_queue" "test" {
  elastic_resource_pool_name = huaweicloud_dli_elastic_resource_pool.test.name
  resource_mode              = 1

  name                  = var.queue_name
  queue_type            = var.queue_type
  cu_count              = var.queue_cu_count
  description           = var.queue_description
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:
- **elastic_resource_pool_name**: The name of the elastic resource pool to which the queue belongs, assigned by referencing the name of the elastic resource pool created above
- **resource_mode**: The resource mode, set to 1 to use the elastic resource pool mode
- **name**: The queue name, assigned by referencing the input variable queue_name
- **queue_type**: The queue type, assigned by referencing the input variable queue_type. The valid values are sql and general
- **cu_count**: The CU count of the queue, assigned by referencing the input variable queue_cu_count
- **description**: The queue description, assigned by referencing the input variable queue_description
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id

### 4. Create a VPC and Subnet

Add the following script in the TF file to create a VPC and subnet:

```hcl
# Create a VPC and subnet in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet. If empty, it is calculated from the VPC CIDR"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet. If empty, it is calculated from the subnet CIDR"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**Parameter Description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The VPC CIDR block, assigned by referencing the input variable vpc_cidr
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC created above
- **subnet name**: The subnet name, assigned by referencing the input variable subnet_name
- **subnet cidr**: The subnet CIDR block, assigned by referencing the input variable subnet_cidr. If empty, it is calculated from the VPC CIDR
- **gateway_ip**: The subnet gateway IP, assigned by referencing the input variable subnet_gateway_ip. If empty, it is calculated from the subnet CIDR

### 5. Create a DLI Enhanced Datasource Connection

Add the following script in the TF file to create a DLI enhanced datasource connection:

```hcl
# Create a DLI enhanced datasource connection in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "datasource_connection_name" {
  description = "The name of the DLI enhanced datasource connection"
  type        = string
}

variable "datasource_connection_hosts" {
  description = "The list of custom hosts for the enhanced datasource connection"

  type = list(object({
    name = string
    ip   = string
  }))

  default  = []
  nullable = false
}

variable "datasource_connection_routes" {
  description = "The list of custom routes for the enhanced datasource connection. Each cidr should be the public destination network to access"

  type = list(object({
    name = string
    cidr = string
  }))
}

resource "huaweicloud_dli_datasource_connection" "test" {
  name      = var.datasource_connection_name
  vpc_id    = huaweicloud_vpc.test.id
  subnet_id = huaweicloud_vpc_subnet.test.id

  dynamic "hosts" {
    for_each = var.datasource_connection_hosts

    content {
      name = hosts.value.name
      ip   = hosts.value.ip
    }
  }

  dynamic "routes" {
    for_each = var.datasource_connection_routes

    content {
      name = routes.value.name
      cidr = routes.value.cidr
    }
  }
}
```

**Parameter Description**:
- **name**: The enhanced datasource connection name, assigned by referencing the input variable datasource_connection_name
- **vpc_id**: The ID of the VPC to which the datasource connection belongs, assigned by referencing the ID of the VPC created above
- **subnet_id**: The ID of the subnet to which the datasource connection belongs, assigned by referencing the ID of the subnet created above
- **hosts**: The custom host information, assigned by referencing the input variable datasource_connection_hosts
- **routes**: The custom route information, assigned by referencing the input variable datasource_connection_routes, where cidr is the public destination network to access

### 6. Associate the Enhanced Datasource Connection with the Elastic Resource Pool

Add the following script in the TF file to associate the enhanced datasource connection with the elastic resource pool:

```hcl
# Associate the enhanced datasource connection with the elastic resource pool in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_dli_datasource_connection_associate" "test" {
  connection_id          = huaweicloud_dli_datasource_connection.test.id
  elastic_resource_pools = [huaweicloud_dli_elastic_resource_pool.test.name]

  depends_on = [huaweicloud_dli_queue.test]
}
```

**Parameter Description**:
- **connection_id**: The enhanced datasource connection ID, assigned by referencing the ID of the enhanced datasource connection created above
- **elastic_resource_pools**: The list of elastic resource pool names to be associated, assigned by referencing the name of the elastic resource pool created above
- **depends_on**: The explicit dependency, ensuring the association is performed after the queue is created

### 7. Create an Elastic IP

Add the following script in the TF file to create an elastic IP:

```hcl
# Create an elastic IP in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "eip_bandwidth_name" {
  description = "The name of the EIP bandwidth"
  type        = string
}

variable "eip_bandwidth_size" {
  description = "The size of the EIP bandwidth in Mbps"
  type        = number
  default     = 5
}

variable "eip_bandwidth_share_type" {
  description = "The share type of the EIP bandwidth"
  type        = string
  default     = "PER"
}

variable "eip_bandwidth_charge_mode" {
  description = "The charge mode of the EIP bandwidth"
  type        = string
  default     = "traffic"
}

resource "huaweicloud_vpc_eip" "test" {
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.eip_bandwidth_name
    size        = var.eip_bandwidth_size
    share_type  = var.eip_bandwidth_share_type
    charge_mode = var.eip_bandwidth_charge_mode
  }
}
```

**Parameter Description**:
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id
- **publicip.type**: The elastic IP type, assigned by referencing the input variable eip_type
- **bandwidth.name**: The bandwidth name, assigned by referencing the input variable eip_bandwidth_name
- **bandwidth.size**: The bandwidth size, assigned by referencing the input variable eip_bandwidth_size
- **bandwidth.share_type**: The bandwidth share type, assigned by referencing the input variable eip_bandwidth_share_type
- **bandwidth.charge_mode**: The bandwidth charge mode, assigned by referencing the input variable eip_bandwidth_charge_mode

### 8. Create a NAT Gateway

Add the following script in the TF file to create a NAT gateway:

```hcl
# Create a NAT gateway in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "nat_gateway_name" {
  description = "The name of the NAT gateway"
  type        = string
}

variable "nat_gateway_spec" {
  description = "The specification of the NAT gateway."
  type        = string
  default     = "1"

  validation {
    condition     = contains(["1", "2", "3", "4"], var.nat_gateway_spec)
    error_message = "The nat_gateway_spec valid value must be `1`, `2`, `3` or `4`."
  }
}

variable "nat_gateway_description" {
  description = "The description of the NAT gateway"
  type        = string
  default     = ""
}

resource "huaweicloud_nat_gateway" "test" {
  name                  = var.nat_gateway_name
  spec                  = var.nat_gateway_spec
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  description           = var.nat_gateway_description
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**Parameter Description**:
- **name**: The NAT gateway name, assigned by referencing the input variable nat_gateway_name
- **spec**: The NAT gateway specification, assigned by referencing the input variable nat_gateway_spec. The valid values are 1, 2, 3, and 4
- **vpc_id**: The ID of the VPC to which the NAT gateway belongs, assigned by referencing the ID of the VPC created above
- **subnet_id**: The ID of the subnet to which the NAT gateway belongs, assigned by referencing the ID of the subnet created above
- **description**: The NAT gateway description, assigned by referencing the input variable nat_gateway_description
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id

### 9. Create an SNAT Rule

Add the following script in the TF file to create an SNAT rule:

```hcl
# Create an SNAT rule in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "snat_description" {
  description = "The description of the SNAT rule"
  type        = string
  default     = ""
}

resource "huaweicloud_nat_snat_rule" "test" {
  nat_gateway_id = huaweicloud_nat_gateway.test.id
  floating_ip_id = huaweicloud_vpc_eip.test.id
  source_type    = 1
  cidr           = huaweicloud_dli_elastic_resource_pool.test.cidr
  description    = var.snat_description
}
```

**Parameter Description**:
- **nat_gateway_id**: The NAT gateway ID, assigned by referencing the ID of the NAT gateway created above
- **floating_ip_id**: The elastic IP ID, assigned by referencing the ID of the elastic IP created above
- **source_type**: The source type, set to 1 to indicate a source CIDR block
- **cidr**: The source CIDR block, assigned by referencing the CIDR block of the elastic resource pool, so that queues in the elastic resource pool can access the public network through SNAT
- **description**: The SNAT rule description, assigned by referencing the input variable snat_description

### 10. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
vpc_name                   = "tf_test_dli_vpc"
vpc_cidr                   = "192.168.0.0/16"
subnet_name                = "tf_test_dli_subnet"
elastic_resource_pool_name = "tf_test_dli_pool"
elastic_resource_pool_cidr = "172.16.0.0/18"
queue_name                 = "tf_test_dli_queue"
datasource_connection_name = "tf_test_dli_conn"

datasource_connection_routes = [
  {
    name = "tf_test_dli_route"
    cidr = "14.17.72.0/24"
  }
]

eip_bandwidth_name = "tf_test_dli_eip_bw"
nat_gateway_name   = "tf_test_dli_nat"
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

### 11. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the DLI queue public network connectivity related resources
4. Run `terraform show` to view the created DLI queue public network connectivity related resources

## Reference Information

- [Huawei Cloud Data Lake Insight Product Documentation](https://support.huaweicloud.com/dli/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DLI Queue Public Network Connectivity](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dli/queue-public-network-connectivity)
