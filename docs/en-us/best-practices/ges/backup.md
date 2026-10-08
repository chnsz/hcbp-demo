# Deploy Graph Backup

## Application Scenario

Graph Engine Service (GES) is a one-stop graph data management and analysis service provided by Huawei Cloud, supporting graph data storage and millisecond-level query and analysis for billions of vertices and edges. It is widely used in scenarios such as social networks, knowledge graphs, financial risk control, and recommendation systems. Graph data carries critical relationship information of your business, and accidental operations or data corruption can directly affect the accuracy of business analysis results.

This best practice will introduce how to use Terraform to automatically deploy a GES graph instance and create a graph backup, including the creation of VPC, subnet, and security group, the configuration of the graph instance, and the creation of the graph backup. By creating a backup for the graph instance, you can restore graph data to a backup point when needed, ensuring the security of graph data and the continuity of your business.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [Security Group (huaweicloud_networking_secgroup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Graph Instance (huaweicloud_ges_graph)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ges_graph)
- [Graph Backup (huaweicloud_ges_backup)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ges_backup)

### Resource/Data Source Dependencies

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_ges_graph
                    └── huaweicloud_ges_backup

huaweicloud_networking_secgroup
    └── huaweicloud_ges_graph
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Virtual Private Cloud

Add the following script in the TF file (such as main.tf) to create a virtual private cloud:

```hcl
# Create a virtual private cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The VPC name for the GES graph"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable vpc_name
- **cidr**: Assigned by referencing the input variable vpc_cidr

### 3. Create a Virtual Private Cloud Subnet

Add the following script in the TF file (such as main.tf) to create a virtual private cloud subnet:

```hcl
# Create a virtual private cloud subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The subnet name for the GES graph"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
}

resource "huaweicloud_vpc_subnet" "test" {
  name       = var.subnet_name
  vpc_id     = huaweicloud_vpc.test.id
  cidr       = var.subnet_cidr
  gateway_ip = var.gateway_ip
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable subnet_name
- **vpc_id**: Assigned by referencing the ID of the virtual private cloud resource
- **cidr**: Assigned by referencing the input variable subnet_cidr
- **gateway_ip**: Assigned by referencing the input variable gateway_ip

### 4. Create a Security Group

Add the following script in the TF file (such as main.tf) to create a security group:

```hcl
# Create a security group resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "security_group_name" {
  description = "The security group name for the GES graph"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable security_group_name

### 5. Create a Graph Instance

Add the following script in the TF file (such as main.tf) to create a graph instance:

```hcl
# Create a graph instance resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "graph_name" {
  description = "The GES graph name"
  type        = string
}

variable "graph_size_type_index" {
  description = "The graph size type index"
  type        = string
  default     = "1"
}

variable "graph_cpu_arch" {
  description = "The CPU architecture type of the GES graph"
  type        = string
  default     = "x86_64"
}

variable "graph_crypt_algorithm" {
  description = "The cryptography algorithm of the GES graph"
  type        = string
}

variable "graph_enable_https" {
  description = "Whether to enable HTTPS for the GES graph"
  type        = bool
  default     = false
}

variable "graph_tags" {
  description = "The key/value pairs to associate with the GES graph"
  type        = map(string)
  default     = {
    key = "val"
    foo = "bar"
  }
  nullable    = false
}

resource "huaweicloud_ges_graph" "test" {
  name                  = var.graph_name
  graph_size_type_index = var.graph_size_type_index
  cpu_arch              = var.graph_cpu_arch
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  crypt_algorithm       = var.graph_crypt_algorithm
  enable_https          = var.graph_enable_https

  tags = var.graph_tags
}
```

**Parameter Description**:
- **name**: Assigned by referencing the input variable graph_name
- **graph_size_type_index**: Assigned by referencing the input variable graph_size_type_index
- **cpu_arch**: Assigned by referencing the input variable graph_cpu_arch
- **vpc_id**: Assigned by referencing the ID of the virtual private cloud resource
- **subnet_id**: Assigned by referencing the ID of the virtual private cloud subnet resource
- **security_group_id**: Assigned by referencing the ID of the security group resource
- **crypt_algorithm**: Assigned by referencing the input variable graph_crypt_algorithm
- **enable_https**: Assigned by referencing the input variable graph_enable_https
- **tags**: Assigned by referencing the input variable graph_tags

### 6. Create a Graph Backup

Add the following script in the TF file (such as main.tf) to create a graph backup:

```hcl
# Create a graph backup resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
resource "huaweicloud_ges_backup" "test" {
  graph_id = huaweicloud_ges_graph.test.id
}
```

**Parameter Description**:
- **graph_id**: Assigned by referencing the ID of the graph instance resource

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through the `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
vpc_name              = "tf_test_ges_vpc"
vpc_cidr              = "192.168.0.0/16"
subnet_name           = "tf_test_ges_subnet"
subnet_cidr           = "192.168.0.0/24"
gateway_ip            = "192.168.0.1"
security_group_name   = "tf_test_ges_secgroup"
graph_name            = "tf_test_ges_graph"
graph_size_type_index = "1"
graph_cpu_arch        = "x86_64"
graph_crypt_algorithm = "generalCipher"
graph_enable_https    = false
graph_tags = {
  key = "val"
  foo = "bar"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the graph backup
4. Run `terraform show` to view the created graph backup

> Note: The creation of the GES graph takes about 30 minutes, and the backup is created after the graph is ready. The backup is automatically deleted when the associated graph is deleted.

## Reference Information

- [Huawei Cloud Graph Engine Service Product Documentation](https://support.huaweicloud.com/ges/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For GES Graph Backup](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ges/backup)
