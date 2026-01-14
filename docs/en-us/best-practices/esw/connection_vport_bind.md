# Deploy Connection with Virtual Port Binding

## Application Scenario

Enterprise Switch (ESW) is a high-performance, highly available enterprise-grade network switching service provided by Huawei Cloud, supporting large Layer 2 network interconnection and achieving Layer 2 network connections across availability zones. Through ESW connection with virtual port binding, you can bind ESW connections with VPC subnet private IPs to achieve flexible configuration and management of network resources. This best practice will introduce how to use Terraform to automatically deploy ESW connection with virtual port binding, including VPC, subnet, ESW instance, ESW connection, VPC subnet private IP, and connection virtual port binding creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [ESW Flavors Data Source (huaweicloud_esw_flavors)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/esw_flavors)

### Resources

- [VPC Resource (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC Subnet Resource (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [ESW Instance Resource (huaweicloud_esw_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/esw_instance)
- [ESW Connection Resource (huaweicloud_esw_connection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/esw_connection)
- [VPC Subnet Private IP Resource (huaweicloud_vpc_subnet_private_ip)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet_private_ip)
- [ESW Connection Virtual Port Binding Resource (huaweicloud_esw_connection_vport_bind)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/esw_connection_vport_bind)

### Resource/Data Source Dependencies

```text
data.huaweicloud_esw_flavors
    └── huaweicloud_esw_instance

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    │   ├── huaweicloud_esw_instance
    │   ├── huaweicloud_esw_connection
    │   └── huaweicloud_vpc_subnet_private_ip
    │       └── huaweicloud_esw_connection_vport_bind
    └── huaweicloud_esw_instance

huaweicloud_esw_instance
    └── huaweicloud_esw_connection
        └── huaweicloud_esw_connection_vport_bind
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query ESW Flavors Data Source

Add the following script to the TF file (e.g., main.tf) to query ESW flavor information:

```hcl
# Query ESW flavor information
data "huaweicloud_esw_flavors" "test" {}
```

**Parameter Description**:
- This data source requires no parameters and will automatically query available ESW flavor information in the current region

### 3. Create VPC Resource

Add the following script to the TF file (e.g., main.tf) to create a VPC:

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

# Create VPC
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: VPC name, assigned by referencing input variable vpc_name
- **cidr**: VPC CIDR block, assigned by referencing input variable vpc_cidr, default value is "192.168.0.0/16"

### 4. Create VPC Subnet Resources

Add the following script to the TF file (e.g., main.tf) to create multiple VPC subnets (this practice requires 2 subnets):

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
}

# Create VPC subnets (create 2 subnets)
resource "huaweicloud_vpc_subnet" "test" {
  count = 2

  vpc_id     = huaweicloud_vpc.test.id
  name       = format("%s-%d", var.subnet_name, count.index)
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**Parameter Description**:
- **count**: Creation count, set to 2, creating 2 subnets
- **vpc_id**: VPC ID, assigned by referencing the VPC resource
- **name**: Subnet name, assigned by referencing input variable subnet_name and count index, format is "subnet_name-0" and "subnet_name-1"
- **cidr**: Subnet CIDR block, assigned by referencing input variables or automatic calculation
- **gateway_ip**: Subnet gateway IP, assigned by referencing input variables or automatic calculation

### 5. Create ESW Instance Resource

Add the following script to the TF file (e.g., main.tf) to create an ESW instance:

```hcl
variable "esw_instance_name" {
  description = "The name of the instance"
  type        = string
  default     = ""
}

variable "esw_instance_ha_mode" {
  description = "The HA mode of the instance"
  type        = string
  default     = ""
}

variable "esw_instance_description" {
  description = "The description of the instance"
  type        = string
  default     = ""
}

variable "esw_instance_tunnel_ip" {
  description = "The tunnel IP of the instance"
  type        = string
  default     = "192.168.0.192"
}

# Create ESW instance
resource "huaweicloud_esw_instance" "test" {
  name        = var.esw_instance_name
  flavor_ref  = try(data.huaweicloud_esw_flavors.test.flavors[0].name, "")
  ha_mode     = var.esw_instance_ha_mode
  description = var.esw_instance_description

  availability_zones {
    primary = try(data.huaweicloud_esw_flavors.test.flavors.0.available_zones[0], "")
    standby = try(data.huaweicloud_esw_flavors.test.flavors.0.available_zones[1], "")
  }

  tunnel_info {
    vpc_id       = huaweicloud_vpc.test.id
    virsubnet_id = huaweicloud_vpc_subnet.test[0].id
    tunnel_ip    = var.esw_instance_tunnel_ip
  }

  charge_infos {
    charge_mode = "postPaid"
  }
}
```

**Parameter Description**:
- **name**: ESW instance name, assigned by referencing input variable esw_instance_name
- **flavor_ref**: Flavor reference, assigned by referencing ESW flavors data source
- **ha_mode**: High availability mode, assigned by referencing input variable esw_instance_ha_mode, optional parameter
- **description**: Instance description, assigned by referencing input variable esw_instance_description, optional parameter
- **availability_zones.primary**: Primary availability zone, assigned by referencing ESW flavors data source
- **availability_zones.standby**: Standby availability zone, assigned by referencing ESW flavors data source
- **tunnel_info.vpc_id**: Tunnel VPC ID, assigned by referencing the VPC resource
- **tunnel_info.virsubnet_id**: Tunnel virtual subnet ID, assigned by referencing the first subnet resource
- **tunnel_info.tunnel_ip**: Tunnel IP, assigned by referencing input variable esw_instance_tunnel_ip, default value is "192.168.0.192"
- **charge_infos.charge_mode**: Charge mode, set to "postPaid" (pay-per-use)

### 6. Create ESW Connection Resource

Add the following script to the TF file (e.g., main.tf) to create an ESW connection:

```hcl
variable "esw_connection_name" {
  description = "The name of the connection"
  type        = string
  default     = ""
}

variable "esw_connection_fixed_ips" {
  description = "The fixed IP IP of the connection"
  type        = list(string)
  default     = []
}

variable "esw_connection_segmentation_id" {
  description = "The segmentation ID of the connection"
  type        = number
  default     = 10000
}

# Create ESW connection
resource "huaweicloud_esw_connection" "test" {
  instance_id  = huaweicloud_esw_instance.test.id
  name         = var.esw_connection_name
  vpc_id       = huaweicloud_vpc.test.id
  virsubnet_id = huaweicloud_vpc_subnet.test[1].id
  fixed_ips    = var.esw_connection_fixed_ips

  remote_infos {
    segmentation_id = var.esw_connection_segmentation_id
    tunnel_ip       = huaweicloud_esw_instance.test.tunnel_info[0].tunnel_ip
    tunnel_port     = huaweicloud_esw_instance.test.tunnel_info[0].tunnel_port
  }
}
```

**Parameter Description**:
- **instance_id**: ESW instance ID, assigned by referencing the ESW instance resource
- **name**: Connection name, assigned by referencing input variable esw_connection_name, optional parameter
- **vpc_id**: VPC ID, assigned by referencing the VPC resource
- **virsubnet_id**: Virtual subnet ID, assigned by referencing the second subnet resource
- **fixed_ips**: Fixed IP list, assigned by referencing input variable esw_connection_fixed_ips, optional parameter
- **remote_infos.segmentation_id**: Segmentation ID, assigned by referencing input variable esw_connection_segmentation_id, default value is 10000
- **remote_infos.tunnel_ip**: Tunnel IP, assigned by referencing the ESW instance's tunnel IP
- **remote_infos.tunnel_port**: Tunnel port, assigned by referencing the ESW instance's tunnel port

### 7. Create VPC Subnet Private IP Resource

Add the following script to the TF file (e.g., main.tf) to create a VPC subnet private IP:

```hcl
# Create VPC subnet private IP
resource "huaweicloud_vpc_subnet_private_ip" "test" {
  subnet_id    = huaweicloud_vpc_subnet.test[1].id
  device_owner = "neutron:VIP_PORT"
}
```

**Parameter Description**:
- **subnet_id**: Subnet ID, assigned by referencing the second subnet resource
- **device_owner**: Device owner, set to "neutron:VIP_PORT" (virtual IP port)

### 8. Create ESW Connection Virtual Port Binding Resource

Add the following script to the TF file (e.g., main.tf) to create ESW connection with virtual port binding:

```hcl
# Create ESW connection virtual port binding
resource "huaweicloud_esw_connection_vport_bind" "test" {
  connection_id = huaweicloud_esw_connection.test.id
  port_id       = huaweicloud_vpc_subnet_private_ip.test.id
}
```

**Parameter Description**:
- **connection_id**: Connection ID, assigned by referencing the ESW connection resource
- **port_id**: Port ID, assigned by referencing the VPC subnet private IP resource

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# VPC and Subnet Configuration
vpc_name    = "tf_test_esw_connection_cport_bind"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_esw_connection_cport_bind"

# ESW Instance Configuration
esw_instance_name    = "tf_test_esw_connection_cport_bind"
esw_instance_ha_mode = "ha"

# ESW Connection Configuration
esw_connection_name            = "tf_test_esw_connection_cport_bind"
esw_connection_segmentation_id = 9999
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially:
   - `esw_instance_ha_mode` can be set to "ha" (high availability mode) or other modes
   - `esw_connection_segmentation_id` needs to be set to segmentation ID for network segmentation identification
   - `esw_connection_fixed_ips` can be set to fixed IP list for the connection, optional parameter
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="esw_instance_name=my_instance" -var="vpc_name=my_vpc"`
2. Environment variables: `export TF_VAR_esw_instance_name=my_instance` and `export TF_VAR_vpc_name=my_vpc`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values. ESW instances need to configure primary and standby availability zones to ensure high availability. ESW connections need to configure remote information, including segmentation ID, tunnel IP, and tunnel port.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create ESW connection with virtual port binding:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating VPC, subnet, ESW instance, ESW connection, VPC subnet private IP, and connection virtual port binding
4. Run `terraform show` to view the details of the created connection virtual port binding

> Note: After the ESW instance is created, you need to wait for the instance status to become available before continuing to create ESW connections. After the ESW connection is created, you can create connection virtual port binding. Ensure that VPC and subnets are correctly created and subnet CIDR is configured correctly.

## Reference Information

- [Huawei Cloud ESW Product Documentation](https://support.huaweicloud.com/esw/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Connection with Virtual Port Binding](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/esw/connection-vport-bind)
