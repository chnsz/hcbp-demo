# Deploy Peering Connection

## Application Scenario

VPC Peering is used to implement private network connectivity between two Virtual Private Clouds (VPCs), suitable for multi-VPC networking, cross-business system communication, and other scenarios. Through VPC Peering, users can achieve secure, low-latency internal network communication between different VPCs, meeting resource access requirements in enterprise multi-network environments. This best practice will introduce how to use Terraform to automatically deploy two VPCs and their subnets, and establish VPC Peering connections and routes.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zone List Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [VPC Peering Connection Resource (huaweicloud_vpc_peering_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_peering_connection)
- [VPC Route Resource (huaweicloud_vpc_route)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_route)

### Resource/Data Source Dependencies

```
data.huaweicloud_availability_zones

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
    └── huaweicloud_vpc_peering_connection
        └── huaweicloud_vpc_route
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Availability Zone Information

Add the following script to the TF file (e.g., main.tf) to query availability zone information (not directly used in this practice, but recommended to keep for future expansion):

```hcl
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- This data source requires no configuration parameters and will automatically obtain all availability zone information in the current region.

### 3. Create VPC and Subnet Resources

Add the following script to the TF file to instruct Terraform to batch create VPCs and their subnets:

```hcl
variable "vpc_configurations" {
  description = "The list of VPC configurations for peering connection"
  type = list(object({
    vpc_name              = string
    vpc_cidr              = string
    subnet_name           = string
    enterprise_project_id = optional(string, null)
  }))
  validation {
    condition     = length(var.vpc_configurations) == 2
    error_message = "Exactly 2 VPC configurations are required for peering connection."
  }
}

# Create VPC resources
resource "huaweicloud_vpc" "test" {
  count = length(var.vpc_configurations)
  name                  = lookup(var.vpc_configurations[count.index], "vpc_name", null)
  cidr                  = lookup(var.vpc_configurations[count.index], "vpc_cidr", null)
  enterprise_project_id = lookup(var.vpc_configurations[count.index], "enterprise_project_id", null)
}

# Create VPC subnet resources
resource "huaweicloud_vpc_subnet" "test" {
  count = length(var.vpc_configurations)
  vpc_id     = huaweicloud_vpc.test[count.index].id
  name       = lookup(var.vpc_configurations[count.index], "subnet_name", null)
  cidr       = try(cidrsubnet(lookup(var.vpc_configurations[count.index], "vpc_cidr", null), 6, 32), null)
  gateway_ip = try(cidrhost(cidrsubnet(lookup(var.vpc_configurations[count.index], "vpc_cidr", null), 6, 32), 1), null)
}
```

**Parameter Description**:
- **vpc_configurations**: List of VPC and subnet configurations, requires exactly 2 sets of configurations
- **name/cidr/enterprise_project_id**: VPC name, CIDR block, enterprise project ID, assigned through vpc_configurations respectively
- **subnet_name/cidr/gateway_ip**: Subnet name, CIDR block, gateway IP, assigned through vpc_configurations and calculation functions respectively

### 4. Create VPC Peering Connection

Add the following script to the TF file to instruct Terraform to create a VPC Peering connection:

```hcl
variable "peering_connection_name" {
  description = "The name of the VPC peering connection"
  type        = string
}

# Create VPC Peering connection
resource "huaweicloud_vpc_peering_connection" "test" {
  count = length(var.vpc_configurations) == 2 ? 1 : 0
  name        = var.peering_connection_name
  vpc_id      = try(huaweicloud_vpc.test[0].id, null) # source VPC
  peer_vpc_id = try(huaweicloud_vpc.test[1].id, null) # target VPC
}
```

**Parameter Description**:
- **name**: Peering connection name, assigned through input variable peering_connection_name
- **vpc_id/peer_vpc_id**: Source VPC and target VPC IDs, referencing the previously created VPC resources respectively

### 5. Configure VPC Peering Routes

Add the following script to the TF file to instruct Terraform to configure Peering routes for each VPC:

```hcl
# Configure VPC Peering routes
resource "huaweicloud_vpc_route" "test" {
  count = length(var.vpc_configurations)
  vpc_id      = huaweicloud_vpc.test[count.index].id
  destination = try(cidrsubnet(lookup(var.vpc_configurations[count.index], "vpc_cidr", null), 6, 33), null)
  type        = "peering"
  nexthop     = try(huaweicloud_vpc_peering_connection.test[0].id, null)
}
```

**Parameter Description**:
- **vpc_id**: VPC ID that the route belongs to, referencing the previously created VPC resource
- **destination**: CIDR block of the peer VPC, automatically calculated
- **type**: Route type, fixed as "peering"
- **nexthop**: Next hop, referencing the ID of the previously created VPC Peering connection

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Huawei Cloud authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC Peering configuration
vpc_configurations = [
  {
    vpc_name    = "tf_test_source_vpc"
    vpc_cidr    = "192.168.0.0/18"
    subnet_name = "tf_test_source_subnet"
  },
  {
    vpc_name    = "tf_test_target_vpc"
    vpc_cidr    = "192.168.128.0/18"
    subnet_name = "tf_test_target_subnet"
  }
]
peering_connection_name = "tf_test_peering"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="peering_connection_name=my-peering"`
2. Environment variables: `export TF_VAR_peering_connection_name=my-peering`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating VPC Peering connections and related resources
4. Run `terraform show` to view the created VPC Peering connection and route details

## Reference Information

- [Huawei Cloud VPC Product Documentation](https://support.huaweicloud.com/vpc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For VPC Peering Connection](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpc/peering)
